# Agile Track Protocol (L1 delta) — CLI-Agnostic Spec

> Extends [../protocol.md](../protocol.md). This document specifies only what the agile track
> changes: the requirements artifact, the execution/scheduling model, gate policy, and the
> rollback cost model. Everything else — role contracts, state-directory conventions, severity
> tiers, deadlock fingerprinting, budget guardrails, the `agent_invoke` adapter contract, CLI
> adapters — is inherited from the root protocol unchanged. Keywords MUST / SHOULD / MAY are
> used in the RFC-2119 sense, as in the root document.

---

## 1. What's inherited vs. what's new

| Root protocol section | Status in this track |
|---|---|
| §4 Role contracts (analyst, architect, developer, qc, security, tester) | **Unchanged.** Same reads/writes/authority per role. |
| §3.2 Role output schemas (`architect-output.json`, `developer-output.json`, `qc-output.json`, `security-output.json`, `tester-output.json`) | **Unchanged**, but now produced *per slice* (see §2) rather than once per feature. |
| §6 Severity & routing | **Unchanged**, applied within each slice's review loop. |
| §7 Loop semantics, caps, deadlock detection | **Unchanged**, applied within each slice. |
| §9 Async signaling | **Unchanged.** |
| §11 `agent_invoke` contract, CLI adapters | **Unchanged.** |
| §3.1 `requirements.json` (flat schema) | **Replaced** by the slice backlog (§3, this document). |
| §5 Workflow (single sequential state machine) | **Replaced** by frontier scheduling over the slice DAG (§4, this document). |
| §8 Milestone gates (three fixed blocking gates) | **Replaced** by per-slice, risk-tiered gates (§5, this document). |
| §10 Rollback & recovery (conservative, one checkpoint model) | **Extended** with the disposability principle and its two exceptions (§6, this document). |
| — | **New:** blast-radius escalation packaging (§7), requirement-revision channel (§8), continuous calibration via `LEARNING.md` (§9). |

---

## 2. Increment sizing — the "abstract feature"

The unit of work in this track is a **slice**: bigger than a single unit-level change, smaller
than an epic. A slice SHOULD satisfy all of:

- **One design decision.** It traces to a single entry in its `architect-output.json` — if
  reviewing it requires holding two unrelated rationales in mind, split it.
- **One demoable capability**, statable in one sentence ("a tenant can set a rate limit"). A
  vertical slice through the stack, not a horizontal layer ("added the rate-limit table").
- **Independently testable** — its own test subset passes without depending on unfinished
  slices.
- **Bounded module footprint** — if it necessarily touches unrelated bounded contexts, it is
  probably two slices glued together.

This sizing exists specifically because the classical heuristic ("what one engineer can do in
a sprint") is calibrated to implementer throughput, and implementer throughput is no longer the
constraint. The correct calibration target is **reviewer bandwidth and context-reconstruction
cost**: a human reviewing an agent's output has no shared planning history to draw on, so the
fixed cost of a review session is dominated by reconstructing intent, not reading the diff. A
slice should be big enough that one review session covers one coherent rationale, not so small
that the human re-derives context every few minutes for fragments.

---

## 3. The slice backlog (replaces `requirements.json`)

The analyst produces a backlog instead of a single flat requirements document.

### 3.1 Schema — `state/backlog.json`

```jsonc
{
  "featureDescription": "string",
  "slices": [
    {
      "id": "slice-01",
      "capability": "one-sentence demoable description",
      "scope": "functional requirements, constraints, success criteria — scoped to this slice",
      "dependsOn": [],                 // slice ids this slice genuinely requires (§4.1)
      "touchesPaths": ["src/main/java/.../RateLimitConfig.java"],  // expected write-path footprint, for incidental-overlap detection (§4.2)
      "riskTier": "routine | novel | irreversible",   // drives gate policy (§5)
      "status": "pending | ready | in_progress | gated | done | invalidated"
    }
  ]
}
```

- `dependsOn` MUST reflect genuine correctness dependency only (slice B cannot be correctly
  implemented without an artifact slice A produces) — not convenience ordering.
- `touchesPaths` is advisory, used only for incidental-overlap detection (§4.2); it does not
  create a scheduling dependency.
- `riskTier` MUST be set by the analyst at backlog creation and MAY be revised by the architect
  once a design decision clarifies the slice's actual blast radius.

### 3.2 Decomposition guidance for the analyst

- Prefer axes that minimize shared interfaces between slices (vertical capability boundaries,
  not shared horizontal layers).
- When a design element is required by many slices (a schema, a shared interface), pull it out
  as its own small **foundation slice** rather than letting every dependent slice redefine it.
  Foundation slices should be flagged (see §4.1) for priority scheduling precisely because many
  slices are blocked on them.
- Every slice MUST be independently reviewable: its `review-summary.md` should not require the
  reader to have the full feature's context memorized.

---

## 4. Execution model — frontier scheduling over a dependency graph

The root track's single sequential state machine (INIT→DESIGN→DRAFT→CODE→TESTS→DONE) is
replaced with a scheduler operating over the slice DAG.

### 4.1 Genuine dependency (`dependsOn`)

A slice is **ready** when every slice in its `dependsOn` has `status: done` (or, for a
non-blocking gate that auto-proceeded, has at least cleared its gate — see §5). The
orchestrator MUST run the full per-slice pipeline (DESIGN → DRAFT → CODE → TESTS, i.e. the
root track's role sequence, scoped to this slice) for every ready slice, up to the WIP limit
(§4.3), rather than processing slices one at a time in submission order.

A slice with unusually high fan-out (many other slices' `dependsOn` name it) is a **foundation
slice**. The orchestrator SHOULD prioritize scheduling and review-summary generation for
foundation slices, and MUST note the fan-out count in that slice's `review-summary.md` so a
human reviewer understands why it's time-sensitive.

### 4.2 Incidental overlap (`touchesPaths`)

Two ready slices whose `touchesPaths` intersect are not logically dependent but MAY collide if
run in parallel (merge conflict, accidental design coupling). The orchestrator SHOULD serialize
only that pair — this is a scheduling hazard, not a backlog dependency, and MUST NOT be
recorded in `dependsOn`.

### 4.3 WIP limit — frontier width, not queue depth

`config.agile.wipLimit` bounds how many slices may be `in_progress` or `gated`
(awaiting a non-blocking timeout or blocking approval) **at once**, counted across the whole
ready frontier — not "how far ahead of the oldest unreviewed slice." This is a deliberate
difference from a linear queue: independent branches of the graph can each carry their own
slack, so a slow review on one branch doesn't consume the whole feature's WIP budget.

The WIP limit is a **static, human-configured guardrail** (set in `config.json`, adjusted
between runs based on `LEARNING.md`), not a value the orchestrator adjusts autonomously at
runtime based on review-debt or any other live signal (§6.1, §9).

---

## 5. Risk-adaptive gates

Every slice, on completing its per-slice CODE/TESTS pipeline, passes through exactly one gate,
whose policy is chosen by `riskTier` (mapped in `config.agile.gatePolicy`):

| `riskTier` | Default gate policy | Rationale |
|---|---|---|
| `routine` | Non-blocking, timeout per `config.agile.gatePolicy.routine.timeoutSeconds` | Isolated, low-novelty change; auto-proceeds if the human doesn't interject, same mechanism as the root track's `IMPLEMENTATION_DRAFT` checkpoint. |
| `novel` | Blocking | First-time design decision, or a shared interface other slices will depend on (foundation slices default here regardless of the analyst's initial tag). |
| `irreversible` | Blocking **and** subject to the side-effect firewall (§6.2) | Crosses a persistence boundary or touches something outside the worktree; must clear a human gate *and* the firewall before any external effect can execute, regardless of gate outcome. |

Gate policy per slice is set from `riskTier` at backlog-creation time and MAY be escalated
(never silently downgraded) by the architect or security role if implementation reveals higher
risk than initially tagged.

---

## 6. The disposability principle and its two exceptions

### 6.1 Disposability (default posture)

Code and tests produced by a slice, and everything downstream of it, are **cheap to discard and
regenerate** as long as they remain inside the feature's git worktree. This licenses a
substantially more permissive rollback posture than the root track's conservative checkpoint
model: on discovering a slice was wrong, the orchestrator SHOULD roll it back and re-run it (and
everything depending on it) without the caution appropriate to human-authored work, because the
dominant cost — compute — is negligible by design.

This does **not** mean cost-free. Two things are explicitly excluded from disposability:

**(a) Human review time.** If a rollback invalidates a slice a human already reviewed (it
cleared a blocking gate, or a non-blocking gate's timeout window had already been observed by a
human), that review attention is spent and cannot be recovered — the human will need to review
the regenerated slice again. The orchestrator MUST increment `reviewDebt` in
`workflow-state.json` whenever this happens:

```jsonc
"reviewDebt": {
  "count": 0,                 // number of previously human-reviewed slices invalidated by rollback, cumulative
  "events": [ { "ts": "...", "sliceId": "...", "reason": "..." } ]
}
```

`reviewDebt` is a **reporting signal only.** It MUST be surfaced in escalation summaries (§7)
and MAY inform how a human sets `config.agile.wipLimit` between runs. It MUST NOT drive any
automatic change to gate policy, WIP limit, or scheduling at runtime — per the design's
governing rule that critical judgment stays with the human (§7).

**(b) Irreversible side effects.** See §6.2.

### 6.2 The side-effect firewall

Disposability holds only for what stays inside the worktree. A slice MUST NOT execute an
action with an effect outside the worktree — an applied database migration against a shared
resource, a live third-party API call, a sent notification, use of production secrets — unless
`config.agile.allowExternalEffects` is explicitly true for that action and the slice has
cleared a blocking gate.

This MUST be enforced structurally, the same way the root track scopes write paths and shell
commands (`config.stack`), not left to prompt discipline: adapters MUST deny any command or
write path classified as externally effectful until the enforcing gate has passed. Default is
deny.

---

## 7. Escalation — blast-radius packaging, not automated judgment

Escalation triggers are unchanged from the root track (CRITICAL issue, `architecturalEscalation`,
iteration-limit-hit, deadlock, budget exhausted) plus one new trigger: a rollback that would
invalidate one or more already-gated slices ("a rollback cascade").

On any escalation, the orchestrator MUST generate a `review-summary.md` that answers, in plain
terms, before asking the human to decide:

- What design assumption or fact changed, and which slice revealed it.
- Which slices are affected downstream (via the dependency graph, §4.1).
- Which of those affected slices already had human eyes on them (cross-referenced with
  `reviewDebt`).
- What, if anything, is irreversible outside the worktree (cross-referenced with the
  side-effect firewall, §6.2).

This packaging exists to reduce the human's cost of reconstructing the situation — **it never
substitutes for the human's decision.** The system computes and displays; it does not decide.
This is a hard design rule, not a default that can be tuned away: no automated policy in this
track may resolve a CRITICAL, architectural, or cascade-triggering escalation without a human
response.

---

## 8. The requirement-revision channel

Any role MAY emit a `requirementRevision` alongside its normal output when it discovers the
backlog itself needs to change (a requirement was ambiguous, a slice's scope was wrong, a new
slice is needed). Unlike an escalation, this is not a stop-the-run event:

- If the revision only affects slices with `status: pending` (not yet started), the
  orchestrator updates `state/backlog.json` and proceeds — no gate, no escalation. This is the
  upward feedback path the root track lacks (its only upward path is
  `architecturalEscalation`, which always stops for a human).
- If the revision affects a slice that is `in_progress`, `gated`, or `done`, it is a rollback
  cascade and MUST go through §6 (disposability + firewall) and §7 (blast-radius escalation) —
  it is not silently applied.

---

## 9. Continuous calibration — `LEARNING.md` as the retrospective

Slice sizing (§2), risk tiering (§5), and the WIP limit (§4.3) are not derived once. They are
expected to be wrong sometimes and to be corrected the way an agile team's retro adjusts
practice, not the way a bug is fixed. At `DONE`, in addition to the root track's learnings, the
orchestrator MUST append to `resources/LEARNING.md`:

- Slices that didn't fit the sizing heuristic (too large to review in one sitting, or so small
  the review's fixed cost dominated).
- Escalations where the blast-radius summary still required the human to reconstruct
  significant extra context (a signal the packaging in §7 needs improvement, not that the human
  needed automation).
- Gate-policy mismatches: a `routine`-tagged slice that should have been blocking, or vice
  versa.
- Notable `reviewDebt` spikes and what caused them.

These entries are inputs to the human adjusting `config.json` before the next run — sizing
heuristics, risk-tier defaults, and the WIP limit are config, not code, precisely so this
recalibration doesn't require touching the protocol itself.

---

## 10. Conformance checklist (agile track)

An adapter/driver conforms to this track iff it also conforms to the root protocol
([../protocol.md §12](../protocol.md)) and additionally:

- [ ] Analyst produces `state/backlog.json` with the schema in §3.1, not a flat
      `requirements.json`.
- [ ] Scheduling runs the ready frontier (§4.1) in parallel up to `config.agile.wipLimit`
      (§4.3), not a linear per-slice queue.
- [ ] Incidental overlaps (§4.2) are serialized without being recorded as `dependsOn` edges.
- [ ] Gate policy is chosen from `riskTier` (§5) and can only be escalated, never silently
      downgraded.
- [ ] Rollback treats worktree-local code/tests as disposable (§6.1) but enforces the
      side-effect firewall (§6.2) as a structural denial, not a prompt instruction.
- [ ] `reviewDebt` is tracked and surfaced in escalations (§6.1, §7) but never used to
      auto-adjust gate policy or WIP limit at runtime.
- [ ] Every escalation's `review-summary.md` includes the blast-radius content in §7.
- [ ] `requirementRevision` on a not-yet-started slice updates the backlog without stopping the
      run (§8); on an already-gated slice it routes through rollback + escalation.
- [ ] `LEARNING.md` is updated at `DONE` with the calibration items in §9.

---

## Revision History

| Date | Change | Reason |
|------|--------|--------|
| 2026-07-01 | Agile track created | Extracted from a design discussion on applying agile principles to multi-agent orchestration: right-sized slices calibrated to reviewer bandwidth (not agent throughput), dependency-graph/frontier scheduling instead of a linear queue, disposability of agent-produced code bounded by human-review-time and side-effect exceptions, and blast-radius-aware escalation that informs but never replaces human judgment. |
