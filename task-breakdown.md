# Task Breakdown

> Part of [Multi-Agent Java Feature Orchestrator](./solution.md)

---

### Task 1: Project Structure and Git Worktree Setup

**Objective**: Create workspace structure with git worktree management script

**Implementation**:
- Create `.kiro/` directory in project root with subdirectories: `agents/`, `prompts/`, `resources/`, `state/`
- Write `scripts/create-feature-worktree.sh` that:
  - Accepts feature name as argument
  - Creates worktree with unique branch name: `agent-feature-{name}-{timestamp}`
  - Copies `.kiro/` configuration into worktree (preserving template state files)
  - Stamps feature name into `workflow-state.json`
  - Worktree path: `../worktrees/agent-feature-{name}-{timestamp}`
- Write `scripts/cleanup-worktree.sh` that:
  - Accepts worktree path as argument
  - Resolves branch name from `git worktree list --porcelain`
  - Removes worktree and associated branch

**Tests**: Verify worktree creation, feature name stamping, and cleanup

---

### Task 2: Shared State and Resources Structure

**Objective**: Define file-based state management for agent communication

**Implementation**:
- Create `.kiro/resources/app-context.md` distilled from project architecture docs:
  - Technology stack table (Java 17, Spring Boot 3.3.x, Spring Cloud, Hibernate, IBM DB2)
  - Full module structure with descriptions
  - Architecture layers, key patterns, external integrations
- Create `.kiro/resources/coding-standards.md` distilled from developer guides:
  - Java 17 standards, Spring Boot patterns, REST conventions
  - Data access patterns, error handling, testing standards
  - Security standards, commit conventions, build commands
- Create `.kiro/resources/LEARNING.md` - empty template for post-workflow learnings
- Create state file templates with full JSON schemas:
  - `workflow-state.json`: featureName, currentMilestone, activeAgent, signal, iterations, budget, contextUsage, issueFingerprints, stashCheckpoints, history
  - `requirements.json`: featureDescription, scope, functionalRequirements, nonFunctionalRequirements, constraints, successCriteria, affectedModules, targetVersions, involvesMigration
  - `architect-output.json`: designDecisions, affectedModules, integrationPoints, newComponents, modifiedComponents, researchNeeded, researchQuery, risks, notes
  - `developer-output.json`: filesChanged, implementationNotes, compilationStatus, researchNeeded, researchQuery, iterationNumber
  - `qc-output.json`: overallStatus, issues (with severity), suggestions, iterationNumber
  - `security-output.json`: overallStatus, vulnerabilities (with severity), recommendations, architecturalEscalation, iterationNumber
  - `tester-output.json`: testsGenerated, testResults (passed/failed/skipped), failures, coverageMetrics, researchNeeded, researchQuery, iterationNumber
  - `researcher-output.json`: query, targetVersion, sources, findings, recommendedApproach
  - `review-summary.md`: empty template

**Tests**: Verify state file schemas are preserved through worktree copy; verify signal script updates `workflow-state.json` correctly

---

### Task 3: Orchestrator Agent Configuration

**Objective**: Create the main orchestrator agent that coordinates workflow

**Key implementation notes**:
- `use_subagent` is NOT included in the default agent toolset — it must be listed explicitly in `tools`
- `toolsSettings` key for subagent config is `subagent` (not `use_subagent`)
- `write` kept out of `allowedTools` — scoped via `toolsSettings.write.allowedPaths`
- `agentSpawn` hook dumps `workflow-state.json` into context on startup
- `use_subagent` requires explicit `agent_name` — without it, defaults to `kiro_default` which is blocked; orchestrator silently does all work itself. Prompt must include invocation syntax example and a hard rule forbidding the orchestrator from doing subagent work.
- `web_search` and `web_fetch` are unavailable in subagent runtime — orchestrator performs all web research directly

**Implementation**:
- Create `.kiro/agents/orchestrator.json`:
  ```json
  {
    "name": "orchestrator",
    "model": "claude-sonnet-4.5",
    "prompt": "file://../prompts/orchestrator-prompt.md",
    "tools": ["read", "write", "shell", "use_subagent", "web_search", "web_fetch", "code"],
    "toolsSettings": {
      "shell": {
        "autoAllowReadonly": true
      },
      "subagent": {
        "availableAgents": ["analyst", "architect", "developer", "qc", "security", "tester"],
        "trustedAgents": ["analyst", "architect", "developer", "qc", "security", "tester"],
        "maxAgentInvocations": 20
      },
      "write": {
        "allowedPaths": [".kiro/state/*.json", ".kiro/state/review-summary.md", ".kiro/resources/LEARNING.md"]
      }
    },
    "resources": [
      "file://../resources/app-context.md",
      "file://../resources/coding-standards.md",
      "file://../resources/LEARNING.md"
    ]
  }
  ```

- Create `.kiro/prompts/orchestrator-prompt.md` covering:
  - Role: workflow orchestrator and decision maker
  - Hard rule: orchestrator must NEVER implement code itself — always delegate to subagents with explicit `agent_name`
  - Subagent invocation syntax example (with `agent_name` field)
  - Pre-flight checks before every subagent invocation: signal check, budget check, counter increment
  - Web research responsibility: orchestrator performs research directly using `web_search`/`web_fetch`; triggers: workflow start (fetch API docs for versions in pom.xml), `researchNeeded: true` from architect/developer/tester, 2+ iterations with similar errors
  - Fresh invocation model: each developer iteration is a new subagent invocation (clean context); state files provide continuity
  - Parallel execution: invoke QC and Security as parallel subagents, merge results before feedback
  - Iteration limits: Developer ↔ QC/Security max 3 iterations; Tester ↔ Developer max 3 iterations
  - Deadlock detection: track issue fingerprints; if same issue reappears after fix, escalate immediately
  - Escalation logic: CRITICAL issues → stop and escalate to human; `architecturalEscalation` flag → re-invoke architect then human review; iteration limit hit → escalate with summary; budget exhausted → escalate with cost summary
  - Git stash checkpoints: `git stash push -m "pre-{agent}-v{iteration}"` before each agent pass; record in `workflow-state.json`
  - Output validation: validate JSON schema of agent outputs before consumption; retry once on malformed output, then escalate
  - Non-blocking checkpoint: after first developer pass, invoke `notify-checkpoint.sh IMPLEMENTATION_DRAFT` (5 min timeout, auto-proceed)
  - Review summary generation: generate `.kiro/state/review-summary.md` before each blocking milestone
  - Milestone tracking: DESIGN_COMPLETE, CODE_COMPLETE, TESTS_PASS
  - Context compaction: run `/compact` after every 5 subagent round-trips
  - Post-workflow: append key learnings to `.kiro/resources/LEARNING.md`

**Tests**: Load agent with `kiro-cli --agent orchestrator`, verify configuration loads and welcome message appears

---

### Task 4: Software Architect Agent Configuration

**Objective**: Create agent for high/mid-level architecture design

**Key implementation notes**:
- `shell.autoAllowReadonly: true` is critical — without it, every `grep`/`find`/`cat` via shell prompts for approval
- Prompt must include explicit tool guidance: `grep`/`glob` are unavailable in subagent runtime; use `code` tool + `shell` workarounds
- `researchNeeded`/`researchQuery` pattern in output schema enables clean orchestrator → research → re-invoke flow

**Implementation**:
- Create `.kiro/agents/architect.json`:
  ```json
  {
    "name": "architect",
    "description": "Designs high and mid-level architecture for new features",
    "model": "claude-sonnet-4.5",
    "prompt": "file://../prompts/architect-prompt.md",
    "tools": ["read", "write", "code", "shell"],
    "allowedTools": ["read", "code"],
    "toolsSettings": {
      "write": {
        "allowedPaths": [".kiro/state/architect-output.json"]
      },
      "shell": {
        "autoAllowReadonly": true
      }
    },
    "resources": [
      "file://../state/requirements.json",
      "file://../state/researcher-output.json",
      "file://../resources/app-context.md",
      "file://../resources/coding-standards.md",
      "file://docs/architecture/**/*.md"
    ],
    "welcomeMessage": "Ready to design architecture. Share the requirements."
  }
  ```

- Create `.kiro/prompts/architect-prompt.md` covering:
  - Role: software architect specializing in Java/Spring Boot enterprise applications
  - Responsibilities: analyze requirements, review existing architecture, propose design
  - Design principles: modularity, maintainability, consistency with existing patterns, minimal changes
  - Research requests: if unfamiliar with a pattern/library, output `researchNeeded: true` with query
  - Tool guidance: use `code` tool for symbol search and code navigation; use `shell` with `grep`/`find` for text search
  - Output format: write JSON to `.kiro/state/architect-output.json`

**Tests**: Invoke architect agent with sample feature request, verify output written to state file

---

### Task 5: Senior Java Developer Agent Configuration

**Objective**: Create agent for implementing code changes

**Key implementation notes**:
- Most resource-heavy agent: reads requirements, architect output, researcher output, QC output, security output, app-context, and coding-standards (7 resources)
- `write.allowedPaths` must include `**/*.xml` for pom.xml and Spring config changes
- `shell.allowedCommands` scoped to `mvn compile` and `mvn test-compile` only — prevents accidental `mvn deploy` or destructive commands
- Prompt must include "feedback loop awareness" section — developer must read QC/security output from previous iterations and address MAJOR issues

**Implementation**:
- Create `.kiro/agents/developer.json`:
  ```json
  {
    "name": "developer",
    "description": "Implements code changes following architectural design",
    "model": "claude-sonnet-4.5",
    "prompt": "file://../prompts/developer-prompt.md",
    "tools": ["read", "write", "shell", "code"],
    "allowedTools": ["read", "code"],
    "toolsSettings": {
      "shell": {
        "allowedCommands": ["mvn compile", "mvn test-compile"],
        "autoAllowReadonly": true
      },
      "write": {
        "allowedPaths": ["src/main/java/**/*.java", "src/main/java/**/*.xml", ".kiro/state/developer-output.json"]
      }
    },
    "resources": [
      "file://../state/requirements.json",
      "file://../state/architect-output.json",
      "file://../state/researcher-output.json",
      "file://../state/qc-output.json",
      "file://../state/security-output.json",
      "file://../resources/app-context.md",
      "file://../resources/coding-standards.md"
    ],
    "welcomeMessage": "Ready to implement. Show me the architecture design."
  }
  ```

- Create `.kiro/prompts/developer-prompt.md` covering:
  - Role: senior Java developer with Spring Boot expertise
  - Responsibilities: implement design, write minimal code, maintain consistency
  - Feedback loop awareness: read `qc-output.json` and `security-output.json` for issues from previous iterations; address MAJOR issues; note iteration number
  - Troubleshooting: if 2+ iterations produce similar errors, output `researchNeeded: true` with error context
  - Tool guidance: use `code` tool for symbol search; use `shell` with `grep`/`find` for text search
  - Output format: write JSON to `.kiro/state/developer-output.json`

**Tests**: Invoke developer agent with architect design, verify code changes and output written to state file

---

### Task 6: Quality Control Agent Configuration

**Objective**: Create agent for code quality review

**Key implementation notes**:
- Uses `claude-haiku-4.5` — QC is pattern-matching work, doesn't need sonnet-level reasoning
- Explicit severity definitions in prompt are essential: without them, agent defaults to vague severity
- `overallStatus` PASS/FAIL rule: PASS only when zero MAJOR + zero CRITICAL; MINOR alone does not cause FAIL

**Implementation**:
- Create `.kiro/agents/qc.json`:
  ```json
  {
    "name": "qc",
    "description": "Reviews code quality, standards compliance, and best practices",
    "model": "claude-haiku-4.5",
    "prompt": "file://../prompts/qc-prompt.md",
    "tools": ["read", "write", "shell", "code"],
    "allowedTools": ["read", "code"],
    "toolsSettings": {
      "shell": {
        "allowedCommands": ["mvn checkstyle:check", "mvn pmd:check"],
        "autoAllowReadonly": true
      },
      "write": {
        "allowedPaths": [".kiro/state/qc-output.json"]
      }
    },
    "resources": [
      "file://../state/developer-output.json",
      "file://../resources/coding-standards.md",
      "file://../resources/app-context.md"
    ],
    "welcomeMessage": "Ready to review code quality."
  }
  ```

- Create `.kiro/prompts/qc-prompt.md` covering:
  - Role: quality control specialist for Java applications
  - Review checklist: naming conventions, error handling, logging, documentation, best practices
  - Tool guidance: use `code` tool for symbol search; use `shell` with `grep`/`find` for text search; use `shell` for `mvn checkstyle:check` and `mvn pmd:check`
  - Issue classification:
    - MINOR: suggestions only, does not block progress
    - MAJOR: blocks CODE_COMPLETE, developer can fix (max 3 loop iterations)
    - CRITICAL: fundamental flaw requiring human decision or architectural change — triggers immediate escalation
  - `overallStatus`: PASS only when zero MAJOR + zero CRITICAL
  - Output format: write JSON to `.kiro/state/qc-output.json`

**Tests**: Invoke QC agent with developer code, verify quality review report written to state file

---

### Task 7: Security Architect Agent Configuration

**Objective**: Create agent for security vulnerability assessment

**Key implementation notes**:
- Uses `claude-sonnet-4.5` (not haiku) — security assessment requires deeper reasoning than QC
- `architecturalEscalation` flag is unique to security agent — triggers architect re-invocation + human review for design-level security flaws
- Prime the agent with app-specific context to focus on PII/data exposure risks

**Implementation**:
- Create `.kiro/agents/security.json`:
  ```json
  {
    "name": "security",
    "description": "Assesses security vulnerabilities and recommends fixes",
    "model": "claude-sonnet-4.5",
    "prompt": "file://../prompts/security-prompt.md",
    "tools": ["read", "write", "shell", "code"],
    "allowedTools": ["read", "code"],
    "toolsSettings": {
      "shell": {
        "allowedCommands": ["mvn dependency-check:check"],
        "autoAllowReadonly": true
      },
      "write": {
        "allowedPaths": [".kiro/state/security-output.json"]
      }
    },
    "resources": [
      "file://../state/developer-output.json",
      "file://../resources/app-context.md"
    ],
    "welcomeMessage": "Ready to assess security."
  }
  ```

- Create `.kiro/prompts/security-prompt.md` covering:
  - Role: security architect specializing in Java/Spring Boot applications
  - Security checklist: OWASP Top 10, SQL injection, XSS, CSRF, authentication, authorization, input validation, data exposure, dependency vulnerabilities
  - Tool guidance: use `code` tool for symbol search; use `shell` with `grep`/`find` for text search; use `shell` for `mvn dependency-check:check`
  - Issue classification:
    - MINOR: suggestions only, does not block progress
    - MAJOR: blocks CODE_COMPLETE, developer can fix (max 3 loop iterations)
    - CRITICAL: fundamental vulnerability requiring human decision — triggers immediate escalation
    - CRITICAL (architectural): requires design change — triggers architect re-invocation + human review; set `architecturalEscalation: true`
  - Output format: write JSON to `.kiro/state/security-output.json`

**Tests**: Invoke security agent with code changes, verify security assessment report written to state file

---

### Task 8: Tester Agent Configuration

**Objective**: Create agent for test generation and execution

**Key implementation notes**:
- Uses `claude-haiku-4.5` — test generation is more formulaic than design/security work
- `write.allowedPaths` must include `**/src/test/**/*.java` — tester can create test files anywhere under test directories
- Include `architect-output.json` as resource — tester needs design intent to write meaningful tests, not just code coverage
- Prompt should instruct agent to discover existing test patterns using `code` tool before generating new tests

**Implementation**:
- Create `.kiro/agents/tester.json`:
  ```json
  {
    "name": "tester",
    "description": "Generates and executes unit and integration tests",
    "model": "claude-haiku-4.5",
    "prompt": "file://../prompts/tester-prompt.md",
    "tools": ["read", "write", "shell", "code"],
    "allowedTools": ["read", "code"],
    "toolsSettings": {
      "shell": {
        "allowedCommands": ["mvn test", "mvn verify"],
        "autoAllowReadonly": true
      },
      "write": {
        "allowedPaths": ["**/src/test/**/*.java", ".kiro/state/tester-output.json"]
      }
    },
    "resources": [
      "file://../state/developer-output.json",
      "file://../state/architect-output.json",
      "file://../resources/app-context.md",
      "file://../resources/coding-standards.md"
    ],
    "welcomeMessage": "Ready to generate and run tests."
  }
  ```

- Create `.kiro/prompts/tester-prompt.md` covering:
  - Role: test engineer specializing in Java/JUnit 5/Spring Test
  - Test patterns: JUnit 5, Mockito, Spring Boot Test, AssertJ
  - Discover existing test patterns using `code` tool before generating new tests
  - Troubleshooting: if 2+ iterations produce similar test failures, output `researchNeeded: true` with error context
  - Tool guidance: use `code` tool for symbol search; use `shell` with `grep`/`find` for text search
  - Output format: write JSON to `.kiro/state/tester-output.json`

**Tests**: Invoke tester agent with implemented code, verify generated tests and test results written to state file

---

### Task 9: Orchestrator Web Research Capability

**Objective**: Provide external knowledge retrieval when internal knowledge is insufficient

**Approach**: No separate researcher agent — `web_search` and `web_fetch` are unavailable in subagent runtime. Orchestrator performs all web research directly in its own context and writes findings to `researcher-output.json` for subagents to consume.

**Implementation**:
- Orchestrator's `tools` array already includes `web_search` and `web_fetch` (from Task 3)
- Orchestrator writes research findings to `.kiro/state/researcher-output.json`:
  ```json
  {
    "query": "",
    "targetVersion": null,
    "sources": [],
    "findings": [],
    "recommendedApproach": ""
  }
  ```
- All subagents that may need research context have `researcher-output.json` in their `resources` array
- Orchestrator prompt (from Task 3) already covers research triggers and protocol

**Tests**: Ask orchestrator to research a Spring Boot topic, verify findings written to state file and readable by subagents

---

### Task 10: Interactive Requirements Gathering Script

**Objective**: Create script to refine feature requirements before starting workflow

**Key implementation notes**:
- There is no `--prompt` flag for `kiro-cli chat` — the initial prompt is passed as a positional argument: `kiro-cli chat "$PROMPT"`
- The requirements prompt must include an explicit hard stop instruction ("Do NOT start implementing. Your ONLY job is to gather and write requirements.") — without it, the default agent continues into implementation when the user says "go ahead"

**Implementation**:
- Create `scripts/gather-requirements.sh` that:
  - Accepts initial feature description and optional worktree path as arguments
  - Invokes `kiro-cli chat "$PROMPT"` with inline prompt for interactive requirements refinement
  - Questions cover: scope, constraints, edge cases, success criteria, affected modules, migration needs
  - Generates `.kiro/state/requirements.json` with full schema
- Usage: `./scripts/gather-requirements.sh "Add user authentication" /path/to/worktree`

**Tests**: Run script with sample feature description, verify `requirements.json` generated with correct schema

---

### Task 11: Milestone Review and Workflow Integration

**Objective**: Integrate all agents with milestone checkpoints into a master workflow script

**Key implementation notes**:
- `kiro-cli` invocation syntax: `kiro-cli chat --agent orchestrator "..."` (not `--prompt`)
- `set -euo pipefail` in `run-feature-workflow.sh` will abort if `gather-requirements.sh` exits non-zero (e.g., user Ctrl+C) — wrap the call with `|| { ... }` to handle gracefully
- `--no-interactive` flag exists on `kiro-cli` for fully automated runs

**Implementation**:
- Create `scripts/notify-checkpoint.sh`:
  - Non-blocking checkpoint with 5 min timeout
  - Shows diff summary, waits for human input
  - Exit code 0 = auto-proceed, exit code 1 = feedback available
- Create `scripts/signal-workflow.sh`:
  - Async pause/abort/continue via `workflow-state.json` signal field
- Create `scripts/review-milestone.sh`:
  - Blocking milestone gate with artifact viewing
  - Continue/retry/abort decisions
- Create `scripts/run-feature-workflow.sh`:
  - Master script: requirements gathering → worktree creation → orchestrator invocation
  - Wraps `gather-requirements.sh` call with `|| { ... }` to handle non-zero exits
- Update `.kiro/prompts/orchestrator-prompt.md` to include:
  - Milestone definitions: DESIGN_COMPLETE, IMPLEMENTATION_DRAFT, CODE_COMPLETE, TESTS_PASS
  - Exact shell commands for each milestone gate and their exit code handling
  - Full bounded iterative workflow logic (as described in Task 3)

**Tests**: Run complete workflow with mock feature, verify milestone reviews trigger at correct points

**Demo**: Full end-to-end feature addition through all milestones
