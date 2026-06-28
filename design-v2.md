# Multi-Agent Feature Orchestrator — v2 (CLI-Agnostic Core)

> Successor to [solution.md](./solution.md) + [task-breakdown.md](./task-breakdown.md).
> The v1 docs remain the authoritative record of the **Kiro CLI** implementation and its
> hard-won learnings. This document re-frames that work as a **portable protocol** with
> thin per-CLI adapters (Kiro, Claude Code, GitHub Copilot CLI), and updates the design
> for current model capabilities.

---

## 0. Why a v2

v1 was a working multi-agent pipeline for adding features to a Java/Spring Boot app, built
on Kiro CLI. Re-reading it, three things stand out:

1. **The core ideas are strong and portable.** Bounded feedback loops with iteration caps,
   three-tier severity routing, fingerprint-based deadlock detection, milestone gates with a
   human in the loop, worktree isolation, and a file-based state bus are all CLI-independent.
   This is, in effect, a hand-rolled **agent harness** with a real **control loop** — built
   before those terms were common.

2. **Most of the document's bulk is Kiro-runtime workarounds**, not design. Subagents could
   not use `web_search`/`web_fetch`/`grep`/`glob`; approvals hung non-interactive subagents;
   `tools`/`allowedTools`/`toolsSettings` interacted subtly. These produced ~70% of the
   "learnings." On other CLIs most of them simply don't apply.

3. **Several mechanisms are now obsolete** because models and harnesses improved: manual
   `/compact` every N round-trips, the "fresh invocation to survive context limits" model,
   and JSON-schema-validate-then-retry as a substitute for reliable structured output.

v2 separates **what is yours** (the protocol) from **what is Kiro's** (the adapter), so the
same pipeline runs on Claude Code or Copilot CLI with a different adapter and almost no
change to the core.

---

## 1. Three-layer architecture

| Layer | Portable? | What lives here |
|-------|-----------|-----------------|
| **L1 — Protocol** | ✅ fully | State schema, role contracts, severity tiers, loop semantics & caps, milestone gates, deadlock detection, escalation rules, budget guardrails. Pure spec — no CLI, no shell. → [protocol.md](./protocol.md) + [config.json](./config.json). |
| **L2 — Driver** | ✅ mostly | Signal/pause/abort, milestone & checkpoint scripts, the run loop. Calls into L3 through a single `agent_invoke` shim. (Worktree *creation* is the orchestrator's own first step — see [protocol.md §5.0](./protocol.md); only manual worktree *cleanup* sits outside it.) |
| **L3 — Adapter** | ❌ per-CLI | Agent definition file format, the "spawn a subagent" primitive, the invocation command, tool/permission mapping. One adapter per CLI. |

```
                ┌─────────────────────────────────────────┐
   human ──────▶│            L2: Driver scripts            │
                │  run · worktree · signal · milestone     │
                └───────────────┬─────────────────────────┘
                                │ agent_invoke(role, input)
                ┌───────────────▼─────────────────────────┐
                │            L3: CLI Adapter               │
                │  kiro │ claude-code │ copilot            │
                └───────────────┬─────────────────────────┘
                                │ reads/writes
                ┌───────────────▼─────────────────────────┐
                │   L1: Protocol state  (state/*.json)     │
                │   the single source of truth + audit log │
                └─────────────────────────────────────────┘
```

The **state directory is the contract** between layers. Any CLI that can read files, write
files, run shell, and spawn a child agent with its own context can host this pipeline.

---

## 2. L1 — The portable protocol

### 2.1 Roles (CLI-independent contracts)

Each role is defined by *inputs it reads*, *outputs it writes*, and *authority* — never by a
vendor config. Adapters render these into vendor agent files.

| Role | Reads | Writes | Authority |
|------|-------|--------|-----------|
| **orchestrator** | all state | `workflow-state.json`, `review-summary.md` | The only human-facing actor; spawns every other role; never edits product code |
| **analyst** | feature request | `requirements.json` | Requirements only; hard stop before implementation |
| **architect** | requirements, research | `architect-output.json` | Design only; may request research |
| **developer** | requirements, design, QC/security/test feedback, research | product code, `developer-output.json` | The only role that writes product code |
| **qc** | dev output, standards | `qc-output.json` | Review only; assigns severity |
| **security** | dev output, design, research | `security-output.json` | Review only; may set `architecturalEscalation` |
| **tester** | dev output, design | test code, `tester-output.json` | Writes test code only |

> Researcher is **not** a separate role in v2. On Kiro it was merged into the orchestrator
> because subagents couldn't web-search; on Claude Code / Copilot any role can be given web
> tools. Treat "research" as a **capability flag** on a role, not a distinct agent (see §4).

### 2.2 State schema (the bus)

Unchanged in spirit from v1, kept as the portable contract. `workflow-state.json` is the
spine:

```jsonc
{
  "featureName": "string",
  "currentMilestone": "DESIGN | DRAFT | CODE | TESTS | DONE",
  "activeRole": "architect",
  "signal": "none | pause | abort",          // written by signal driver, polled pre-spawn
  "iterations": { "dev_review": 0, "dev_test": 0 },
  "budget": { "maxInvocations": 30, "used": 0 },
  "issueFingerprints": ["sha256:..."],         // deadlock detection
  "stashCheckpoints": ["pre-developer-v1"],
  "history": [ { "ts": "...", "role": "...", "event": "..." } ]
}
```

Role outputs keep their v1 schemas (`requirements.json`, `architect-output.json`, …). The
one schema change: every review output carries `issues[]` with a stable `fingerprint`
(hash of normalized issue text + location) so the orchestrator can detect repeats.

### 2.3 Control structures (the "loop")

These are the heart of the design and are fully portable:

- **Bounded feedback loops.** `developer ↔ {qc, security}` and `developer ↔ tester`, each
  capped (default 3). On cap hit → escalate, never silently continue.
- **Three-tier severity.** `MINOR` (log, continue) · `MAJOR` (loop back to developer) ·
  `CRITICAL` (stop, escalate to human immediately). `security` additionally sets
  `architecturalEscalation: true` to re-invoke the architect before the human.
- **Deadlock detection.** If a fingerprint reappears after a developer "fix," escalate
  immediately instead of burning the remaining iteration budget. (This is the single most
  valuable mechanism in v1 — most "loop" implementations lack it.)
- **Milestone gates.** Blocking: `DESIGN_COMPLETE`, `CODE_COMPLETE`, `TESTS_PASS`.
  Non-blocking: `IMPLEMENTATION_DRAFT` (timed auto-proceed).
- **Budget guardrail.** Hard cap on total role invocations; exhaustion escalates with a
  cost summary.
- **Rollback.** A checkpoint (git stash or commit) before each developer pass; on abort the
  worktree is preserved for manual recovery.

### 2.4 Parallelism

QC and security are independent reviews → run concurrently, merge results before feeding the
developer. Portable: Kiro `use_subagent` (≤4 parallel), Claude Code parallel `Task` calls,
Copilot Fleet mode.

---

## 3. The harness & loop — formalized

You asked me to scrutinize the "AI Harness" and "Loop" angle. Mapping v1 onto current
terms, and noting what to keep vs. retire:

| Concept | v1 mechanism | Verdict in v2 |
|---------|--------------|----------------|
| **Sandbox** | one git worktree per feature | **Keep**, but the orchestrator now creates it itself via shell (no external `create-feature-worktree.sh`); cleanup stays manual. |
| **Permissioning** | `allowedTools` + `toolsSettings.write.allowedPaths` + `autoApprove` | **Keep the intent, move to adapter.** Scoped write paths and auto-approve policy are portable concepts; the exact keys are Kiro's. |
| **Context management** | manual `/compact` every 5 round-trips; "fresh invocation" each loop | **Retire as a survival tactic.** Larger context windows + native compaction make this unnecessary. *Keep* fresh-context-per-review where it improves **objectivity** (a reviewer with a clean slate is a feature, not a workaround). |
| **Message bus** | `state/*.json` + manual schema validation + retry | **Keep the bus; relax the validation.** Reliable tool/structured output makes validate-then-retry mostly redundant; keep a light shape-check at gates. |
| **Control loop** | bounded iterations, severity routing, deadlock, escalation | **Keep — this is the crown jewel.** Promote to L1 spec. |
| **Async control** | `signal-workflow.sh` writes a `signal` field, polled pre-spawn | **Keep.** Clean, portable pause/abort. |
| **Human-in-the-loop** | milestone scripts; 5-min non-blocking checkpoint | **Keep.** Make the timeout configurable, not hard-coded. |

Bottom line: your "primitive harness" was real and most of its *structure* survives. What
gets deleted is the part that was compensating for 2025-era model/runtime limits, not the
part that encodes your engineering judgment.

---

## 4. CLI capability matrix (drives the adapters)

| Capability | Kiro CLI | Claude Code | Copilot CLI |
|------------|----------|-------------|-------------|
| Agent def format | `.kiro/agents/*.json` | `.claude/agents/*.md` (YAML frontmatter) | `*.agent.md` (YAML frontmatter) |
| Subagent primitive | `use_subagent` tool | `Task` tool | subagent / Fleet mode |
| Own context per subagent | ✅ | ✅ | ✅ |
| Parallel subagents | ✅ (≤4) | ✅ | ✅ (Fleet) |
| **Web search in subagent** | ❌ (orchestrator only) | ✅ (WebSearch/WebFetch) | ✅ |
| **grep/glob in subagent** | ❌ (use `shell`) | ✅ (Grep/Glob) | ✅ |
| Per-agent model select | ✅ `model` | ✅ `model` | ✅ |
| Lifecycle hooks | ✅ | ✅ (settings.json hooks) | ⚠️ via MCP/config |
| Invocation | `kiro-cli chat --agent N "…"` | `Task`/headless `claude -p` | `copilot --agent N --prompt "…"` |

The two ✅ cells where Kiro had ❌ are why ~half of v1's "learnings" evaporate on the other
two CLIs: no Researcher-merge, no `shell grep` workaround, no `researchNeeded` round-trip
required (it becomes an optimization, not a necessity).

---

## 5. The adapter contract (L2 ↔ L3)

Every adapter implements one function, conceptually:

```
agent_invoke(role, input_text, { parallel?: role[], model?, web?: bool }) -> output_state_written
```

Driver scripts never name a CLI. They call `agent_invoke`; a single env var
(`ORCH_CLI=kiro|claude|copilot`) selects the adapter. Each adapter provides:

1. **`render-agents`** — generate the vendor agent files from the L1 role contracts.
2. **`invoke`** — spawn one role (and optionally a parallel set), blocking until state is written.
3. **`map-permissions`** — translate scoped write paths + auto-approve into vendor config.

### 5.1 Kiro adapter

Essentially v1, unchanged. `.kiro/agents/*.json`, orchestrator owns web research, subagents
get `code` + `shell grep` workarounds, `shell.autoApprove: true` on non-interactive
subagents, `write` in `tools` + `allowedPaths` + `fallbackAction: "deny"`. All the v1
learnings (solution.md §"Workflow Integration Learnings") are the Kiro adapter's notes.

### 5.2 Claude Code adapter

```markdown
---
name: developer
description: Implements code changes following the architectural design.
tools: Read, Edit, Write, Bash, Grep, Glob          # WebSearch/WebFetch if web:true
model: sonnet
---
<role prompt body — same contract as L1 developer>
```

- Subagents are markdown in `.claude/agents/`, spawned via the **Task tool**. The
  orchestrator is itself a subagent (or the top-level session) that issues parallel `Task`
  calls for qc+security.
- **Drop** the Researcher-merge: give `architect`/`security`/`tester` `WebSearch` directly
  when `web:true`. `researchNeeded` becomes an optional optimization, not a workaround.
- Permissions via `settings.json` `permissions.allow`/`deny` + hooks (e.g. a `PostToolUse`
  hook running `mvn spotless:apply` on edited Java — the v1 "future enhancement," now native).
- Milestone gates as **slash commands** or driver scripts shelling `claude -p` headless.
- Context: rely on native compaction; no manual `/compact` cadence.

### 5.3 Copilot CLI adapter

```markdown
---
name: developer
description: Implements code changes following the architectural design.
tools: [read, edit, shell, search]
mcp-servers: { ... }
---
<role prompt body — same contract as L1 developer>
```

- `*.agent.md` files at repo or user level; invoke with `copilot --agent developer --prompt "…"`.
- Use **Fleet mode** for the parallel qc+security phase.
- Subagents carry their own context window (good for the clean-reviewer property).
- Hooks are thinner; lean on driver scripts + MCP for lifecycle actions.

---

## 6. Model selection (updated)

v1 used `claude-sonnet-4.5` / `claude-haiku-4.5` (dotted Kiro naming, now stale). Current
lineup and recommended assignment:

| Role | Model | ID | Why |
|------|-------|----|----|
| orchestrator | Opus 4.8 | `claude-opus-4-8` | Owns workflow logic, routing, escalation judgment |
| architect | Opus 4.8 | `claude-opus-4-8` | Deep design reasoning |
| security | Opus 4.8 | `claude-opus-4-8` | Highest-stakes review; subtle reasoning |
| developer | Sonnet 4.6 | `claude-sonnet-4-6` | Strong code generation at lower cost than Opus |
| qc | Haiku 4.5 | `claude-haiku-4-5-20251001` | Pattern-matching / standards checks |
| tester | Haiku 4.5 | `claude-haiku-4-5-20251001` | Formulaic test generation |
| analyst | Haiku 4.5 | `claude-haiku-4-5-20251001` | Structured requirements extraction |

Notes:
- Adapters map these IDs to each CLI's accepted form (Claude Code accepts aliases like
  `opus`/`sonnet`/`haiku`; Kiro historically used dotted strings — verify per CLI).
- Consider **extended/interleaved thinking** for orchestrator, architect, and security; it
  materially helps routing and design decisions and was not available when v1 was written.
- Cost/iteration caps (§2.3) matter more with Opus on three roles — keep the budget guardrail.

---

## 7. Migration from v1

1. **Lift** the control-loop + severity + deadlock + milestone logic out of
   `orchestrator-prompt.md` into an L1 spec doc — done: [protocol.md](./protocol.md), with
   all tunables in [config.json](./config.json). All adapters import these.
2. **Quarantine** the Kiro-specific learnings as the Kiro adapter's README; they stay true
   for that adapter, false for the others.
3. **Delete from the core**: manual `/compact` cadence, `shell grep/find` tool guidance,
   orchestrator-does-all-research, JSON-validate-retry-as-control-flow.
4. **Reconcile the v1 contradiction**: solution.md:152 ("`use_subagent` *is* in the default
   agent") vs task-breakdown.md:62 ("it is *not*"). Correct statement: it's in the default
   toolset, **but** an explicit `tools` array *replaces* defaults, so it must be relisted.
   This belongs only in the Kiro adapter notes.
5. **Update model IDs** everywhere (§6).
6. **Parameterize** hard-coded constants: iteration caps, the 5-minute checkpoint, budget —
   move to a `config.json` consumed by L2.

---

## 8. Open questions / future

- **Single worktree vs. parallel feature worktrees.** v1 assumes one feature at a time.
  Driver could manage a pool for concurrent features.
- **Native scheduling/background.** Claude Code background tasks and Copilot cloud agents
  could run the tester or a full suite asynchronously (v1's unused `delegate` idea, now
  first-class on two CLIs).
- **Metrics.** Emit per-loop iteration counts, tokens/role, time-to-milestone from L2 —
  trivial once L2 owns `agent_invoke`.
- **Generality beyond Java.** The protocol is language-agnostic; only `resources/` and the
  `shell.allowedCommands` (mvn/checkstyle/…) are Java-specific. A `stack-profile` in config
  would let the same core target other ecosystems.

---

## Revision History

| Date | Change | Reason |
|------|--------|--------|
| 2026-06-28 | v2 created | Re-architected v1 into CLI-agnostic protocol (L1) + driver (L2) + per-CLI adapters (L3) for Kiro, Claude Code, Copilot CLI. Updated model IDs to Opus 4.8 / Sonnet 4.6 / Haiku 4.5. Formalized harness/loop concepts; retired Kiro-era workarounds from the core. |

---

### Sources

- [Create custom subagents — Claude Code Docs](https://code.claude.com/docs/en/sub-agents)
- [Creating and using custom agents for GitHub Copilot CLI — GitHub Docs](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/create-custom-agents-for-cli)
- [Custom agents and sub-agent orchestration — GitHub Docs](https://docs.github.com/en/copilot/how-tos/copilot-sdk/use-copilot-sdk/custom-agents)
- [Kiro Agent Configuration Reference](https://kiro.dev/docs/cli/custom-agents/configuration-reference/)
