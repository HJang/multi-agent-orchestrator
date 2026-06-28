# Orchestrator Protocol (L1) — CLI-Agnostic Spec

> The portable core of the [Multi-Agent Feature Orchestrator](./design-v2.md). This document
> defines **behavior and contracts only** — no CLI, no shell syntax, no vendor config. Every
> adapter (Kiro, Claude Code, Copilot CLI) MUST implement this spec. Tunable values live in
> [config.json](./config.json) and are referenced here as `config.<path>`.
>
> Keywords MUST / SHOULD / MAY are used in the RFC-2119 sense.

---

## 1. Scope

This protocol coordinates specialized AI roles to add a feature to a codebase, with bounded
feedback loops, severity-based routing, deadlock detection, human-in-the-loop milestone
gates, and rollback. It is language-agnostic; the only stack-specific data is isolated in
`config.stack`.

It deliberately says nothing about *how* a role is spawned or *what file format* defines it.
That is the adapter's job (§11, the `agent_invoke` contract).

---

## 2. Vocabulary

- **Role** — a specialized actor with a fixed input/output/authority contract (§4). Realized
  by an adapter as a vendor agent.
- **Invocation** — one spawn of one role with a clean context, terminating when it has
  written its output to the state directory.
- **State directory** — `config.feature.stateDir`; the single source of truth and audit log.
- **Orchestrator** — the one role that talks to the human, spawns other roles, and owns the
  control loop. It MUST NOT edit product code.
- **Milestone** — a named checkpoint (§8). Blocking milestones require human approval;
  non-blocking ones auto-proceed on timeout.
- **Fingerprint** — a stable hash of a normalized issue (text + location) used for deadlock
  detection (§7).

---

## 3. State directory contract

All inter-role communication happens through files in `config.feature.stateDir`. Roles
communicate **only** through these files — never through return values, stdout, or shared
memory. This makes the bus CLI-independent and gives a complete audit trail.

### 3.1 `workflow-state.json` (the spine)

```jsonc
{
  "featureName": "string",
  "currentMilestone": "INIT | DESIGN | DRAFT | CODE | TESTS | DONE",
  "activeRole": "string | null",
  "signal": "none | pause | abort",       // written by the async signal driver, §9
  "iterations": { "dev_review": 0, "dev_test": 0 },
  "budget": { "maxInvocations": 30, "used": 0 },
  "issueFingerprints": ["sha256:..."],
  "stashCheckpoints": ["pre-developer-v1"],
  "history": [ { "ts": "ISO-8601", "role": "string", "event": "string" } ]
}
```

### 3.2 Role outputs

One file per role, each a JSON document. Required fields below; adapters/prompts MAY add
more. Every **review** output (`qc`, `security`) MUST carry `issues[]` where each issue has
`{ id, severity, location, description, fingerprint }`.

| File | Producer | Required fields |
|------|----------|-----------------|
| `requirements.json` | analyst | `featureDescription, scope, functionalRequirements, nonFunctionalRequirements, constraints, successCriteria, affectedModules` |
| `architect-output.json` | architect | `designDecisions, affectedModules, integrationPoints, newComponents, modifiedComponents, risks, researchNeeded, researchQuery` |
| `developer-output.json` | developer | `filesChanged, implementationNotes, compilationStatus, iterationNumber, researchNeeded, researchQuery` |
| `qc-output.json` | qc | `overallStatus, issues[], suggestions, iterationNumber` |
| `security-output.json` | security | `overallStatus, issues[], recommendations, architecturalEscalation, iterationNumber` |
| `tester-output.json` | tester | `testsGenerated, testResults{passed,failed,skipped}, failures[], iterationNumber, researchNeeded, researchQuery` |
| `researcher-output.json` | (capability) | `query, sources, findings, recommendedApproach` |
| `review-summary.md` | orchestrator | free text shown to human at gates |

### 3.3 Output validity

Before consuming a role's output, the orchestrator MUST shape-check it (required fields
present, JSON parses). On failure: retry the invocation **once**, then escalate. This is a
*light* check — it is not a substitute for the role doing its job, and adapters on CLIs with
reliable structured output MAY make it a no-op beyond JSON parsing.

---

## 4. Role contracts

Each role is defined by what it reads, what it writes, and its authority. Adapters render
these into vendor agent definitions; the contract is invariant across CLIs.

| Role | Reads | Writes | Authority |
|------|-------|--------|-----------|
| **orchestrator** | all state | `workflow-state.json`, `review-summary.md` | Human-facing; spawns roles; owns the loop. **No product-code edits.** |
| **analyst** | feature request | `requirements.json` | Requirements only. **Hard stop before implementation.** |
| **architect** | requirements, research | `architect-output.json` | Design only. May set `researchNeeded`. |
| **developer** | requirements, design, prior qc/security/test outputs, research | product code, `developer-output.json` | **Only role that writes product code.** Writes restricted to `config.stack.writePaths.developer`. |
| **qc** | developer output, standards | `qc-output.json` | Review only; assigns severity. |
| **security** | developer output, design, research | `security-output.json` | Review only; MAY set `architecturalEscalation`. |
| **tester** | developer output, design | test code, `tester-output.json` | Writes test code only (`config.stack.writePaths.tester`). |

Authority is enforced by the adapter's permission mapping (scoped write paths, allowed
commands per `config.stack`).

---

## 5. Workflow (the control loop)

State machine driven by the orchestrator. Every transition is appended to
`workflow-state.json.history`.

```
INIT
  └─ spawn analyst ─▶ requirements.json
       └─ present to human ──(refine)──▶ re-spawn analyst
                          └─(approve)─▶ DESIGN

DESIGN
  └─ [research if config.research] ─▶ spawn architect ─▶ architect-output.json
       └─ gate: DESIGN_COMPLETE (blocking) ─▶ DRAFT

DRAFT
  └─ checkpoint (rollback §10) ─▶ spawn developer ─▶ developer-output.json
       └─ gate: IMPLEMENTATION_DRAFT (non-blocking, timed) ─▶ CODE
            └─(human intervenes in window)─▶ apply feedback, re-spawn developer

CODE
  └─ spawn qc ∥ security  (parallel) ─▶ merge outputs
       ├─ CRITICAL?            ─▶ escalate human (§8)
       ├─ architecturalEscalation? ─▶ re-spawn architect, then human
       ├─ MAJOR & iter ≤ cap?  ─▶ loop to developer (dev_review++)
       ├─ MAJOR & iter > cap?  ─▶ escalate ITERATION_LIMIT_HIT
       └─ clean?               ─▶ gate: CODE_COMPLETE (blocking) ─▶ TESTS

TESTS
  └─ spawn tester ─▶ tester-output.json
       ├─ failures & iter ≤ cap? ─▶ loop to developer (dev_test++)
       ├─ failures & iter > cap? ─▶ escalate ITERATION_LIMIT_HIT
       └─ pass?                  ─▶ gate: TESTS_PASS (blocking) ─▶ DONE

DONE
  └─ append learnings to resources, workflow complete
```

### 5.1 Pre-invocation checks (MUST run before every spawn)

1. **Signal check** — read `workflow-state.json.signal`; if `abort`, stop and preserve
   worktree; if `pause`, wait until cleared.
2. **Budget check** — if `budget.used >= budget.maxInvocations`, escalate per
   `config.budget.onExhausted`.
3. **Increment** — `budget.used++`, set `activeRole`, append to `history`.

---

## 6. Severity & routing

Every review issue carries a severity. Routing is defined by `config.severity`:

| Severity | Meaning | Action |
|----------|---------|--------|
| `MINOR` | suggestion | log and continue (does not block a gate) |
| `MAJOR` | must fix, fixable by developer | loop back to developer (within cap) |
| `CRITICAL` | fundamental flaw / vulnerability | stop, escalate to human immediately |

`overallStatus` for a review is `PASS` **iff** zero MAJOR and zero CRITICAL. MINOR alone does
not fail a review.

Security-only: a design-level flaw sets `architecturalEscalation: true`, which re-spawns the
architect (a design revision) **before** human escalation, rather than looping the developer.

---

## 7. Loop semantics, caps & deadlock

- Each feedback loop has an iteration counter in `workflow-state.json.iterations` and a cap
  in `config.loops.<name>.maxIterations`.
- On reaching the cap, perform `config.loops.<name>.onCap` (default `escalate`). Never
  silently continue past a cap.
- **Deadlock detection** (`config.deadlock`): when a review produces an issue whose
  `fingerprint` already exists in `workflow-state.json.issueFingerprints`, the same problem
  has survived a developer "fix." Escalate **immediately** rather than spending remaining
  iterations. New fingerprints are appended each round.

Fingerprint = `config.deadlock.fingerprintAlgo` over a normalized string of issue
description + file location (normalize whitespace and line numbers so cosmetic diffs don't
change the hash).

---

## 8. Milestone gates

- **Blocking** (`config.milestones.blocking`): `DESIGN_COMPLETE`, `CODE_COMPLETE`,
  `TESTS_PASS`. The orchestrator generates `review-summary.md`, presents it to the human, and
  waits. Human decision: continue / retry / abort.
- **Non-blocking** (`config.milestones.nonBlocking`): `IMPLEMENTATION_DRAFT` notifies the
  human and waits `timeoutSeconds`; on timeout it performs `onTimeout` (default `proceed`).
  If the human responds within the window, their feedback re-spawns the developer.

Every gate MUST be preceded by a fresh `review-summary.md` sized for a few-minute human read.

---

## 9. Async signaling

A side channel lets the human pause/abort mid-run without racing the orchestrator: the
signal driver writes `workflow-state.json.signal` ∈ {`none`,`pause`,`abort`}. The
orchestrator polls it in the pre-invocation check (§5.1). Roles never read the signal — only
the orchestrator acts on it, between invocations, keeping state transitions atomic.

---

## 10. Rollback & recovery

Before each role listed in `config.rollback.checkpointBefore` (default: `developer`), the
orchestrator creates a checkpoint via `config.rollback.mechanism` (default `git_stash`),
named `pre-<role>-v<iteration>`, and records it in `stashCheckpoints`. On abort, the worktree
is preserved (`config.rollback.preserveWorktreeOnAbort`) so the human can inspect or restore
any checkpoint manually.

---

## 11. The `agent_invoke` contract (L2 ↔ adapter)

Adapters expose exactly one operation to the driver:

```
agent_invoke(role, inputText, options) -> void   // blocks until role output is written
```

- `role` — one of §4.
- `inputText` — the initial message (the orchestrator's instruction for this invocation).
- `options` — `{ parallel?: role[], model?: string, web?: boolean }`.
  - `parallel` present → spawn the given roles concurrently, return when all have written.
  - `model` → overrides `config.models.<role>`.
  - `web` → grant web tools to the role (honored where the CLI supports it; on CLIs without
    subagent web tools, the adapter routes research through the orchestrator instead — see
    `config.research.inlineWebTools`).

The driver selects the adapter from `config.cli` (or env `ORCH_CLI`) and otherwise never
references a specific CLI.

Each adapter additionally provides:
- **render-agents** — generate vendor agent files from the §4 role contracts + `config`.
- **map-permissions** — translate authority (write paths, allowed commands) into vendor
  permission config.

---

## 12. Conformance checklist

An adapter conforms iff:

- [ ] Spawns each §4 role with its declared inputs, outputs, and authority.
- [ ] Enforces scoped write paths and allowed commands from `config.stack`.
- [ ] Runs qc and security in parallel and merges before routing.
- [ ] Implements the §5 state machine and §5.1 pre-invocation checks.
- [ ] Honors severity routing (§6), loop caps and deadlock escalation (§7).
- [ ] Implements blocking and non-blocking gates (§8) with `review-summary.md`.
- [ ] Polls the signal channel (§9) and supports pause/abort.
- [ ] Creates and records rollback checkpoints (§10).
- [ ] Reads all tunables from `config.json`; hard-codes none of them.

---

## Revision History

| Date | Change | Reason |
|------|--------|--------|
| 2026-06-28 | Extracted L1 protocol from design-v2 | Split pure spec from architecture narrative; parameterized all constants into config.json. |
