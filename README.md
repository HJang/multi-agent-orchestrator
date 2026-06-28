# Multi-Agent Feature Orchestrator

A CLI-agnostic system that coordinates specialized AI agents to add a feature to a codebase
end-to-end — requirements → design → implementation → review → tests — with bounded feedback
loops, severity-based routing, deadlock detection, and human-in-the-loop checkpoints.

It started as a Kiro-CLI-specific design ([solution.md](./solution.md),
[task-breakdown.md](./task-breakdown.md)) and was re-architected into a portable protocol
([design-v2.md](./design-v2.md)) that runs on **Claude Code**, **GitHub Copilot CLI**, or
**Kiro CLI** through thin adapters.

> **Status:** the protocol and configuration are complete and authoritative
> ([protocol.md](./protocol.md), [config.json](./config.json)). The per-CLI **adapters**
> (agent files + driver scripts) are specified but not yet implemented. The per-CLI sections
> below describe how each CLI realizes the protocol and how it will be invoked.

---

## Document map

| File | What it is |
|------|------------|
| [README.md](./README.md) | This file — overview, concepts, per-CLI instructions |
| [design-v2.md](./design-v2.md) | The architecture: three layers, capability matrix, model selection, migration plan |
| [protocol.md](./protocol.md) | The portable L1 spec — behavior & contracts only (RFC-2119) |
| [config.json](./config.json) | All tunables — loop caps, models, budget, stack-specific commands |
| [orchestrator-guide.md](./orchestrator-guide.md) | **Step-by-step: how to create the orchestrator agent** on each CLI |
| [prompts/orchestrator.md](./prompts/orchestrator.md) | The portable orchestrator system prompt (one block customized per CLI) |
| [solution.md](./solution.md) | **v1** — the original Kiro-specific design + implementation learnings |
| [task-breakdown.md](./task-breakdown.md) | **v1** — task-by-task implementation plan |

---

## What design-v2 achieves

1. **One pipeline, three CLIs.** The same workflow — the seven roles, the loops, the gates —
   runs on Claude Code, Copilot CLI, or Kiro CLI. Switching is a one-line change
   (`config.cli`). v1 was welded to Kiro.

2. **A clean separation of concerns** across three layers:
   - **L1 Protocol** (portable) — *what* happens: state schema, role contracts, severity
     routing, loop caps, deadlock detection, milestone gates, escalation, rollback.
   - **L2 Driver** (mostly portable) — *the run loop*: worktree lifecycle, signaling,
     milestone scripts. Calls one `agent_invoke(role, input)` shim and never names a CLI.
   - **L3 Adapter** (per-CLI) — *how* a role is spawned: agent file format, subagent
     primitive, invocation command, permission mapping.

3. **Policy pulled out of behavior.** Every constant — iteration caps, the checkpoint
   timeout, budget, model assignments, and all Java/Maven-specific commands — lives in
   [config.json](./config.json). Retargeting to another language is a single `stack` block.

4. **Current-generation models.** Opus 4.8 on the reasoning-heavy roles (orchestrator,
   architect, security), Sonnet 4.6 on the developer, Haiku 4.5 on the high-volume roles
   (qc, tester, analyst), with extended thinking enabled where it pays off.

5. **A conformance checklist** ([protocol.md §12](./protocol.md)) so any new adapter can be
   verified against the spec rather than against one reference implementation.

---

## Challenges & gaps filled from the original design

| v1 problem | How v2 resolves it |
|------------|--------------------|
| **Vendor lock-in.** Every mechanism was expressed in Kiro config (`use_subagent`, `toolsSettings`, `.kiro/agents/*.json`). | Behavior moved to a vendor-neutral protocol; Kiro specifics demoted to one adapter. |
| **Workarounds dominated the design.** ~Half of v1's "learnings" existed only because Kiro subagents couldn't use `web_search`/`web_fetch`/`grep`/`glob` and hung on approval prompts. | Claude Code and Copilot subagents have these tools natively. The Researcher-merge, the `shell grep` substitutions, and the `researchNeeded` round-trip become optional optimizations, not necessities. |
| **Obsolete context tactics.** Manual `/compact` every 5 round-trips and "fresh invocation to survive context limits." | Retired from the core — current context windows + native compaction handle it. Fresh-context-per-review is *kept*, but reframed as a reviewer-objectivity feature, not a survival hack. |
| **Validate-then-retry as control flow.** JSON schema validation + retry substituted for unreliable structured output. | Relaxed to a light shape-check at gates; reliable tool output makes the heavy machinery redundant. |
| **Hard-coded constants** scattered through prompts (3 iterations, 5-minute timeout, 20-invocation budget, model IDs). | Centralized in [config.json](./config.json); the protocol references them by key. |
| **Stale model IDs** (`claude-sonnet-4.5`, `claude-haiku-4.5`, dotted naming). | Updated to `claude-opus-4-8` / `claude-sonnet-4-6` / `claude-haiku-4-5-20251001`. |
| **Internal contradiction.** solution.md:152 says `use_subagent` *is* in the default toolset; task-breakdown.md:62 says it is *not*. | Reconciled: it *is* in defaults, **but** an explicit `tools` array replaces defaults, so it must be relisted. This now lives only in the Kiro adapter notes. |
| **Java-specific assumptions** baked throughout. | Isolated to `config.stack`; the rest of the protocol is language-agnostic. |

---

## Harness & Loop — how this design interprets them

This project independently arrived at two ideas that later became common vocabulary. v2 names
them explicitly.

### The Harness

The **harness** is everything around the model that turns a chat completion into a reliable
worker. Here it is the union of:

- **Sandbox** — one git worktree per feature, isolating agent work from the main branch.
- **Permissioning** — scoped write paths and allowed commands per role (`config.stack`), so
  the developer can touch `src/main/**` but not delete your build, and reviewers can't write
  code at all. Authority is part of each role contract ([protocol.md §4](./protocol.md)).
- **Message bus** — the state directory. Roles communicate *only* through `state/*.json`,
  which doubles as a complete audit log.
- **Async control** — a signal channel the human can use to pause/abort mid-run, polled
  between invocations so state transitions stay atomic.
- **Guardrails** — a hard invocation budget and rollback checkpoints before risky passes.

In v1 this harness was hand-built and Kiro-shaped. In v2 it is specified abstractly (L1) and
realized per-CLI (L3) — the *concept* is portable, the *plumbing* is the adapter's job. The
parts of v1 that were compensating for 2025-era runtime limits are removed; the parts that
encode engineering judgment are promoted to the spec.

### The Loop

The **loop** is the control structure that lets agents iterate toward "done" without running
away. This design's loop is more disciplined than a plain retry-until-pass:

- **Bounded feedback loops** — `developer ↔ {qc, security}` and `developer ↔ tester`, each
  with an iteration cap. Hitting the cap **escalates**; it never silently continues.
- **Severity-based routing** — `MINOR` logs and continues, `MAJOR` loops back to the
  developer, `CRITICAL` stops and escalates immediately. Security can additionally route a
  design-level flaw back to the **architect** before the human.
- **Deadlock detection** — every issue gets a stable fingerprint; if the same fingerprint
  reappears after a "fix," the loop is stuck, so it escalates *immediately* instead of
  burning the remaining budget. This is the single most valuable mechanism in the design and
  is what most "agent loops" lack.
- **Milestone gates** — blocking human checkpoints at design/code/tests, plus one
  non-blocking timed checkpoint after the first draft.

In short: the harness makes each agent *safe to run*, and the loop makes the *collection of
agents converge* — with explicit exits when they can't.

---

## Running it per CLI

All three share the same flow: pick the CLI in [config.json](./config.json), then launch the
orchestrator from the repo root with a natural-language feature description. The orchestrator
creates its own feature worktree and drives everything else.

> **Building the orchestrator agent itself?** Follow the
> [step-by-step orchestrator guide](./orchestrator-guide.md) — it walks through the system
> prompt, the per-CLI agent definition, launch, and verification.

```jsonc
// config.json
"cli": "claude"   // or "copilot" or "kiro"
```

> The agent files and driver scripts referenced below are produced by each adapter
> (`render-agents` / driver) and are **not yet committed**. The commands show the intended
> invocation shape once an adapter is implemented.

### Claude Code  *(recommended reference adapter — cleanest fit)*

- **Agents:** `.claude/agents/<role>.md`, each with YAML frontmatter (`name`, `description`,
  `tools`, `model`) generated from the L1 role contracts. Reviewers get `Read, Grep, Glob`;
  the developer gets `Read, Edit, Write, Bash`; research-capable roles get
  `WebSearch, WebFetch`.
- **Subagents:** spawned via the **Task tool**. The orchestrator issues parallel `Task` calls
  for the qc + security phase.
- **Permissions:** `settings.json` `permissions.allow`/`deny` mirror `config.stack` write
  paths and allowed commands; a `PostToolUse` hook can auto-format edited files.
- **Run:** (the orchestrator creates its own feature worktree as its first step — no setup script)
  ```bash
  #    config.json -> "cli": "claude"
  # launch from the repo root; headless drives the loop, omit -p for interactive
  claude -p "Add per-tenant API rate limiting" --agent orchestrator
  ```
- **Notes:** no Kiro-style workarounds needed — web tools, Grep/Glob, and compaction are
  native. Disk-defined subagents require a session restart to load (or create them via the
  `/agents` UI).

### GitHub Copilot CLI

- **Agents:** `<role>.agent.md` files (YAML frontmatter: `name`, `description`, `tools`,
  `mcp-servers`) at repository or user level.
- **Subagents:** each runs in its own context window; use **Fleet mode** for the parallel
  qc + security phase.
- **Permissions:** tool list per agent; secrets/MCP via Agents secrets or repo/org config.
- **Run:** (the orchestrator creates its own feature worktree as its first step — no setup script)
  ```bash
  #    config.json -> "cli": "copilot"
  # launch from the repo root
  copilot --agent orchestrator --prompt "Add per-tenant API rate limiting"
  ```
- **Notes:** hooks are thinner than Claude Code — lean on MCP for lifecycle actions. Naming
  conflicts resolve user > repo > org > enterprise.

### Kiro CLI  *(the original v1 implementation)*

- **Agents:** `.kiro/agents/<role>.json` with `tools`/`allowedTools`/`toolsSettings`, exactly
  as in [solution.md](./solution.md) and [task-breakdown.md](./task-breakdown.md).
- **Subagents:** `use_subagent` (must be relisted in an explicit `tools` array; ≤4 parallel).
- **Permissions:** `write` in `tools` + `toolsSettings.write.allowedPaths` +
  `fallbackAction: "deny"`; `shell.autoApprove: true` on non-interactive subagents.
- **Web research:** subagents can't web-search — the **orchestrator** performs research and
  writes `researcher-output.json` (set `config.research.inlineWebTools: false`).
- **Run:** (the orchestrator creates its own feature worktree as its first step — no setup script)
  ```bash
  #    config.json -> "cli": "kiro"
  # launch from the repo root; initial prompt is POSITIONAL (no --prompt flag)
  kiro-cli chat --agent orchestrator "Add per-tenant API rate limiting"
  ```
- **Notes:** all of v1's learnings apply to this adapter and only this adapter. The initial
  prompt is a **positional** argument (there is no `--prompt` flag).

---

## Quick start (conceptual)

1. Choose a target CLI and set `config.cli`.
2. Adjust `config.stack` for your language/build (defaults target Java + Maven).
3. Review the loop caps, budget, and checkpoint timeout in `config.json`.
4. Launch the orchestrator from the repo root with your feature description — it creates the
   feature worktree itself, then drives the pipeline.
5. Respond at the milestone gates (design, code, tests); the orchestrator handles the loops,
   routing, and escalation in between.

---

## License

See [LICENSE](./LICENSE).
