# Multi-Agent Feature Orchestrator — Agile Track

A second execution model for the same orchestrator, sitting alongside the root
[design-v2.md](../design-v2.md) / [protocol.md](../protocol.md) track. Where the root track runs
one feature through a single sequential pass (requirements → design → implement → review →
test, with bounded fix-loops *within* implementation), this track decomposes a feature into a
**backlog of small, mostly-independent vertical slices**, schedules them as a dependency graph
so unrelated work never waits on a slow human review, and treats agent-produced code as cheap
to discard — bounded by two things that are *not* cheap: human review attention and irreversible
side effects.

This document set exists on its own because it changes the *shape* of the workflow, not just a
parameter of it. It does not replace the root track.

---

## Which track to use

| | Root track (design-v2 / protocol.md) | Agile track (this directory) |
|---|---|---|
| **Unit of work** | The whole feature, one pass | A backlog of small vertical slices |
| **Requirements artifact** | One `requirements.json` | A slice backlog with an explicit dependency graph |
| **Execution order** | Strictly sequential: DESIGN → DRAFT → CODE → TESTS | Frontier-scheduled: any slice whose dependencies have cleared runs now, in parallel with others |
| **Feedback loops** | Sideways only — developer ↔ qc/security/tester, within implementation | Sideways (same) **plus** upward — any role can revise not-yet-started backlog entries |
| **Milestone gates** | Three fixed blocking gates for the whole feature + one non-blocking checkpoint | Per-slice, risk-tiered: routine slices non-blocking, novel/irreversible slices blocking |
| **Rollback posture** | Conservative — checkpoint before developer, preserve on abort | Aggressive for code/tests (disposable, cheap to redo); strict for anything with external side effects |
| **Best fit** | Small or well-understood features; a single coherent design that fits in one pass; teams that want the simpler model | Larger or uncertain features where requirements are likely to shift as slices land; when you want agents kept busy in parallel instead of idling behind one review |

Both tracks share the same foundations: role contracts (analyst/architect/developer/qc/security/
tester), the state-directory contract, severity tiers (MINOR/MAJOR/CRITICAL), deadlock
fingerprinting, the CLI adapters, and model selection. **This track only replaces the workflow,
scheduling, and gating layer** — see [protocol.md](./protocol.md) §1 for exactly what's inherited
vs. what's new.

---

## Core principles

1. **Increment size is set by human review bandwidth, not agent implementation speed.**
   With human engineers, implementer throughput and reviewer throughput were roughly coupled —
   sprint-sized work was naturally reviewer-sized work. AI agents decouple that: implementation
   is no longer the constraint, so sizing has to be chosen for the reviewer, explicitly. The
   right unit is an **abstract feature** — bigger than a single unit-level change, smaller than
   an epic — sized by "one design decision, one demoable capability," not by effort estimate.

2. **Minimize genuine dependency between slices; expose what remains as an explicit graph.**
   Some slices really do need another slice's output — that can't be designed away. What can be
   controlled is *how much* depends on any one slice, and whether the pipeline treats
   dependency as a linear queue (everything waits on the slowest review) or a graph (only what
   actually depends on the slow item waits).

3. **Gate policy is risk-adaptive, not uniform.** Routine, isolated slices default to
   non-blocking with a timeout (the same pattern as the root track's `IMPLEMENTATION_DRAFT`
   checkpoint, generalized). Novel design decisions, first-time shared-interface changes, or
   anything crossing a persistence/external boundary default to blocking.

4. **Agent-produced code and tests are cheap and disposable — within the worktree.** Rolling
   back a slice and regenerating everything downstream of it costs agent-compute, which is
   nearly free compared to human rework. This licenses a much more permissive rollback posture
   than classical software engineering assumes.

5. **...but two things are explicitly not disposable, and the design must protect them:**
   - **Human review time already spent.** A rollback that invalidates a slice a human already
     reviewed doesn't just discard code — it discards attention, and forces a repeat. This is
     tracked (not throttled) as `reviewDebt`, purely to keep it visible.
   - **Irreversible side effects outside the worktree.** Anything that touches a real external
     system (an applied schema migration, a live API call, a sent notification) is not undone
     by a rollback. This is enforced structurally as a side-effect firewall, not left to agent
     discipline.

6. **The system's job is to make blast radius legible, not to decide for the human.** Hard
   calls — a CRITICAL issue, a design assumption breaking, a rollback cascade — still go to a
   human. The improvement is that the escalation is packaged with what changed, what's
   affected, what already had eyes on it, and what's irreversible, so the human isn't
   reconstructing that from raw state under time pressure. Nothing here auto-decides based on
   these signals; the human always does.

7. **This is a continuous practice, not a solved formula.** Slice sizing, risk tiering, and WIP
   limits are not derived once and fixed — they're recalibrated run over run, the same way an
   agile team's retro adjusts sprint practice rather than declaring a final answer.
   `LEARNING.md` is that retrospective mechanism here.

---

## Document map

| File | What it is |
|------|------------|
| [protocol.md](./protocol.md) | The agile-track L1 spec: slice backlog schema, dependency-graph scheduling, risk-adaptive gates, the disposability principle and its two exceptions, blast-radius escalation packaging, and the continuous-calibration role of `LEARNING.md`. |
| [config.json](./config.json) | Agile-specific tunables: WIP limit (frontier-based), gate-policy-by-risk-tier mapping, side-effect firewall flags, review-debt tracking toggle. Overlays the root [config.json](../config.json); everything not listed here (models, severity, deadlock, stack) is inherited unchanged. |
| [prompts/orchestrator.md](./prompts/orchestrator.md) | The orchestrator system prompt for this track: backlog decomposition, frontier scheduling, risk-tiered gating, disposability-aware rollback, blast-radius summary generation. |

Role prompts for analyst/architect/developer/qc/security/tester are **not duplicated** here —
the root track's role contracts ([protocol.md §4](../protocol.md)) apply unchanged; only the
analyst's *output shape* changes (a backlog instead of a flat document), which is specified in
this track's [protocol.md](./protocol.md) §3.

---

## Where this came from

This track is the direct result of pushing on one question: agile methodology wasn't invented
because many engineers need to collaborate — it was invented because waterfall's big upfront
requirements/design diverge from reality before feedback arrives, and by the time feedback
does arrive the system is too committed to cheaply change. There's no reason that lesson stops
applying just because the implementer is now an AI agent instead of a human team. But naively
importing agile's human-calibrated mechanisms (sprint-sized stories, synchronous PR review)
doesn't work either, because the thing agile implicitly assumed — implementer and reviewer
throughput are roughly matched — breaks the moment the implementer is much faster than the
reviewer. This track is what's left after working through that: right-sized increments,
dependency-graph scheduling instead of a linear queue, disposability as the new cost model,
and a hard line that judgment on critical calls still belongs to a human — the system's job is
only to make that judgment cheaper to exercise, never to replace it.
