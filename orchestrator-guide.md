# How to Create the Orchestrator Agent — Step by Step

The orchestrator is the brain of the pipeline: it talks to the human, spawns every other
role, and runs the control loop. This guide takes you from nothing to a launchable
orchestrator on **Claude Code**, **GitHub Copilot CLI**, or **Kiro CLI**.

You need three things, in order:

1. **The system prompt** — the orchestrator's behavior. Already written and portable:
   [`prompts/orchestrator.md`](./prompts/orchestrator.md). You only customize one block.
2. **The agent definition** — a small CLI-specific file that points at the prompt, grants
   tools, sets the model, and lists which subagents it may spawn.
3. **The launch command.**

> Prerequisite: just a git repo and `config.json`. The orchestrator creates the feature
> worktree and seeds the state directory itself as its first step ([protocol.md §5.0](./protocol.md)) —
> there is no `create-feature-worktree.sh` and you do not pre-create `state/`. You only need
> templates for the state files available for it to copy. Throughout, `state/` =
> `config.feature.stateDir` (relative to the worktree).

---

## Step 1 — Prepare the system prompt (all CLIs)

1. Copy [`prompts/orchestrator.md`](./prompts/orchestrator.md) into place (or reference it
   where it is).
2. Open it and find the block titled **"How you spawn a role ⟨ADAPTER-SPECIFIC⟩"**.
3. Replace the commented examples with the **one** mechanism for your CLI (the exact text is
   in each CLI section below). Leave the rest of the prompt unchanged — it already encodes the
   full workflow, severity routing, loops, deadlock detection, and escalation from
   [protocol.md](./protocol.md).

That single edit is the only change the orchestrator prompt needs to target a CLI.

---

## Step 2 — Create the agent definition

Pick your CLI.

### A) Claude Code

1. Create `.claude/agents/orchestrator.md`:

   ```markdown
   ---
   name: orchestrator
   description: Coordinates the multi-agent feature pipeline; the only human-facing role.
   tools: Read, Write, Bash, Task, WebSearch, WebFetch
   model: opus
   ---
   <paste the full contents of prompts/orchestrator.md here,
    OR keep them in prompts/orchestrator.md and paste them in —
    Claude Code agent bodies are inline, so include the prompt text directly>
   ```

   - `Task` is what lets the orchestrator spawn the other roles as subagents.
   - `WebSearch`/`WebFetch` let it do research directly (and subagents can have them too).
   - `model: opus` → maps to `config.models.orchestrator` (`claude-opus-4-8`).

2. Fill the **spawn block** in the prompt with:

   > Use the **Task tool**. Set the subagent type to the role name and put your instruction in
   > the prompt. For the parallel review phase, issue **two Task calls in the same turn** —
   > one `qc`, one `security` — so they run concurrently.

3. Create the six subagent files too (`.claude/agents/{analyst,architect,developer,qc,security,tester}.md`)
   from their [protocol.md §4](./protocol.md) contracts. The orchestrator can only spawn roles
   that exist as agent files.

4. Permissions: in `.claude/settings.json`, mirror `config.stack` — allow the developer's
   write paths and the per-role `allowedCommands`, deny everything else. Optionally add a
   `PostToolUse` hook to auto-format edited files.

   > Disk-defined agents load on session start. After creating the files, restart the session
   > (or use `/agents` to create them live).

### B) GitHub Copilot CLI

1. Create `orchestrator.agent.md` (repo-level: `.github/agents/`, or user-level):

   ```markdown
   ---
   name: orchestrator
   description: Coordinates the multi-agent feature pipeline; the only human-facing role.
   tools: [read, write, shell, search]
   ---
   <paste the full contents of prompts/orchestrator.md here>
   ```

   - Copilot resolves the model from your CLI/account configuration; map the intended
     `config.models.orchestrator` there.
   - `mcp-servers:` may be added in the frontmatter if a role needs MCP tools.

2. Fill the **spawn block** in the prompt with:

   > Spawn the subagent whose name matches the role. For the parallel review phase, use
   > **Fleet mode** to run `qc` and `security` concurrently.

3. Create the six role agents as `*.agent.md` files from their
   [protocol.md §4](./protocol.md) contracts.

4. Permissions: scope each agent's `tools` and configure secrets/MCP via Agents secrets at
   repo/org level. Naming conflicts resolve user > repo > org > enterprise.

### C) Kiro CLI  (matches the original v1 implementation)

1. Create `.kiro/agents/orchestrator.json`:

   ```json
   {
     "name": "orchestrator",
     "model": "claude-opus-4-8",
     "prompt": "file://../prompts/orchestrator.md",
     "tools": ["read", "write", "shell", "use_subagent", "web_search", "web_fetch", "code"],
     "toolsSettings": {
       "shell": { "autoAllowReadonly": true },
       "subagent": {
         "availableAgents": ["analyst", "architect", "developer", "qc", "security", "tester"],
         "trustedAgents":   ["analyst", "architect", "developer", "qc", "security", "tester"],
         "maxAgentInvocations": 30
       },
       "write": {
         "allowedPaths": ["state/*.json", "state/review-summary.md", "resources/LEARNING.md"]
       }
     },
     "resources": [
       "file://../resources/app-context.md",
       "file://../resources/coding-standards.md"
     ]
   }
   ```

   Kiro gotchas (from v1 — these are real and only apply here):
   - `use_subagent` **must** be in the explicit `tools` array — an explicit array replaces
     the defaults rather than extending them.
   - The `toolsSettings` key is `subagent`, not `use_subagent`.
   - Keep `write` in `tools` (so `allowedPaths` is live) but **out** of any `allowedTools`,
     and rely on `allowedPaths` for scoping.
   - `file://` paths resolve relative to the JSON file's own directory, so use `../` to climb
     out of `agents/`.

2. Fill the **spawn block** in the prompt with:

   > Use the `use_subagent` tool with an explicit `agent_name`, e.g.
   > `use_subagent(agent_name="developer", prompt="...")`. It supports up to 4 parallel
   > subagents — spawn `qc` and `security` together. **web_search/web_fetch are unavailable to
   > subagents**, so YOU do all web research and write `state/researcher-output.json`.

   Also set `config.research.inlineWebTools: false` so the protocol routes research through
   the orchestrator.

3. Create the six subagent JSON files from their [protocol.md §4](./protocol.md) contracts,
   each with `shell.autoApprove: true` (non-interactive subagents hang on approval prompts
   otherwise) and `allowedTools` for read/code so they don't prompt on every tool use.

---

## Step 3 — Launch

```bash
# Claude Code
claude -p "Add per-tenant API rate limiting" --agent orchestrator

# Copilot CLI
copilot --agent orchestrator --prompt "Add per-tenant API rate limiting"

# Kiro CLI   (initial prompt is POSITIONAL — there is no --prompt flag)
kiro-cli chat --agent orchestrator "Add per-tenant API rate limiting"
```

Launch from the **repo root**. The orchestrator's first action is to create the feature
worktree itself (`git worktree add …`) and seed `state/` inside it; then it spawns `analyst`,
presents requirements for your approval, and drives design → draft → review → tests, pausing
only at the milestone gates.

---

## Step 4 — Verify it's wired correctly

Quick checks before trusting a full run:

- [ ] The agent loads without config errors and prints its role.
- [ ] On launch it **creates a worktree** under `config.feature.worktreeRoot` and records
      `worktreePath`/`branch` in `workflow-state.json` (`git worktree list` shows it). If it
      skips this, confirm the orchestrator has a shell tool (`Bash`/`shell`) and the SETUP
      section is present in the prompt.
- [ ] On a trivial request it **spawns `analyst`** rather than answering directly. If it
      starts implementing itself, the spawn block or the "HARD RULES" weren't applied — recheck
      Step 1.
- [ ] After analyst returns, `state/requirements.json` exists and is valid JSON.
- [ ] It pauses at **DESIGN_COMPLETE** and waits for your input (gate is working).
- [ ] `state/workflow-state.json` shows `budget.used` incrementing and `history` growing.
- [ ] Writing `"signal": "pause"` into `workflow-state.json` makes it stop before the next
      spawn; `"none"` resumes.

---

## Common pitfalls

| Symptom | Cause | Fix |
|---------|-------|-----|
| Orchestrator writes code itself | Spawn block empty or HARD RULES not in prompt | Re-apply Step 1; confirm the prompt body is actually loaded |
| First spawn silently fails / it does all the work | Role name not passed explicitly (esp. Kiro `agent_name`) | Use the exact spawn syntax from your CLI's section |
| Subagent hangs forever | Non-interactive subagent waiting on an approval prompt | Kiro: `shell.autoApprove: true` + `allowedTools`; Claude/Copilot: pre-allow the tools |
| "Model not available" / silent fallback | Wrong model ID | Use the IDs in `config.models` (`claude-opus-4-8`, etc.) |
| Research never happens on Kiro | Expecting subagents to web-search | Set `inlineWebTools: false`; orchestrator does research |
| Loop never stops | Caps not read / deadlock check skipped | Confirm the prompt's CODE/TESTS sections and `config.loops` are intact |

---

## What to build next

Repeat Step 2 for the six role agents (same pattern: contract from
[protocol.md §4](./protocol.md) → CLI agent file → scoped permissions from `config.stack`).
The orchestrator is non-functional until at least `analyst` and `developer` exist, since it
delegates everything.
