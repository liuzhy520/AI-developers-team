# Workflow Guide

A detailed walkthrough of the multi-agent orchestration workflow for VS Code GitHub Copilot.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Workflow Phases](#workflow-phases)
  - [Phase 1: Initialization](#phase-1-initialization)
  - [Phase 2: Planning](#phase-2-planning)
  - [Phase 3: Task Creation](#phase-3-task-creation)
  - [Phase 4: Execution](#phase-4-execution)
  - [Phase 5: Testing](#phase-5-testing)
  - [Phase 6: Bug Handling](#phase-6-bug-handling)
  - [Phase 7: Completion](#phase-7-completion)
- [State Management](#state-management)
- [Parallel Execution](#parallel-execution)
- [Contract Tasks](#contract-tasks)
- [Example Walkthrough](#example-walkthrough)

---

## Overview

The multi-agent orchestration system uses four specialized roles to deliver software changes:

```
User Request → Orchestrator → Planner → Task Cards → Executor(s) → Tester(s) → Done
                    ↑                                                    │
                    └──────────── Bug Re-assignment ←────────────────────┘
```

Each role has strict boundaries:
- **Orchestrator**: Manages state, dispatches agents. Only writer of `.ai-control/`.
- **Planner**: Plans tasks. Read-only, no code changes.
- **Executor**: Implements one task. Scoped to allowed paths, one branch per task.
- **Tester**: Tests and reports. Read-only for business code.

## Prerequisites

1. VS Code with GitHub Copilot extension installed.
2. This repository cloned locally.
3. Agent modes visible in Copilot Chat (the `.github/agents/*.agent.md` files define them).

## Workflow Phases

### Phase 1: Initialization

The Orchestrator reads or creates the workflow state.

**If starting fresh:**

1. User opens Copilot Chat, selects **@Orchestrator**.
2. User describes the request (e.g., "Build a REST API for user management").
3. Orchestrator creates:
   - `.ai-control/state.json` — from `templates/state.json`
   - `.ai-control/prfaq.md` — requirement summary
   - `.ai-control/task-board.md` — empty task board
   - `.ai-control/bug-board.md` — empty bug board

**If resuming:**

1. Orchestrator reads `.ai-control/state.json`.
2. Checks `task-board.md` and `bug-board.md` for current status.
3. Determines next action based on state.

### Phase 2: Planning

The Orchestrator dispatches the Planner subagent.

```
runSubagent(
  agentName="Planner",
  prompt="You are the Planner subagent. Goal: <user request>. Read .ai-control/prfaq.md and the repository. Return task breakdown, dependencies, acceptance criteria, and test recommendations."
)
```

The Planner returns:
- Task breakdown with IDs, titles, allowed paths
- Dependency graph
- Parallelization advice
- Test recommendations
- Risks and blockers

The Orchestrator persists the plan to `.ai-control/plan.md`.

### Phase 3: Task Creation

The Orchestrator creates task cards from the Planner's output.

For each task:
1. Copy `.ai-control/templates/TASK-template.md` to `.ai-control/tasks/TASK-NNN.md`.
2. Fill in: `id`, `title`, `owner`, `branch`, `worktree`, `allowed_paths`, `depends_on`, `acceptance`, `verification`.
3. Update `state.json` with the new task.
4. Update `task-board.md` with the new row.

### Phase 4: Execution

The Orchestrator dispatches Executor subagents for tasks whose dependencies are satisfied.

```
runSubagent(
  agentName="Executor",
  prompt="You are an Executor. Task card: .ai-control/tasks/TASK-001.md. Branch: feat/TASK-001-user-model. Worktree: ../wt-TASK-001. Read the task card, implement, verify, commit, push, and return structured output."
)
```

**Executor workflow:**
1. Read task card → extract allowed paths, acceptance, verification.
2. Create branch and worktree:
   ```bash
   git worktree add ../wt-TASK-001 -b feat/TASK-001-user-model
   ```
3. Implement within allowed paths.
4. Run verification commands.
5. Commit and push.
6. Return structured handoff.

**After executor returns:**
- Orchestrator persists handoff to `.ai-control/handoffs/HANDOFF-TASK-001.md`.
- Updates task status to `ready_for_test` in `state.json` and `task-board.md`.
- If executor returned `blocked`, investigates and replans.

### Phase 5: Testing

The Orchestrator dispatches the Tester subagent.

```
runSubagent(
  agentName="Tester",
  prompt="You are the Tester. Task card: .ai-control/tasks/TASK-001.md. Branch: feat/TASK-001-user-model. Commit: abc1234. Run verification and regression tests. Return structured report."
)
```

**Tester workflow:**
1. Check out the target branch/commit.
2. Run verification commands from the task card.
3. Run any additional tests from the test plan.
4. Produce evidence (command output, logs).
5. If failures found, draft bug cards.
6. Return structured test report.

**After tester returns:**
- Orchestrator persists report to `.ai-control/reports/REPORT-TASK-001.md`.
- If `passed` → mark task `done`.
- If `failed` → create bug cards, enter bug handling flow.

### Phase 6: Bug Handling

When a tester reports failures:

1. Orchestrator creates bug cards in `.ai-control/bugs/BUG-NNN.md`.
2. Updates `bug-board.md`.
3. Decides resolution strategy:
   - **Reassign** to original executor with updated criteria.
   - **New task** if the fix requires different scope.
   - **Won't fix** if out of scope.
4. Dispatches the appropriate executor with the bug context.
5. After fix, dispatches tester for retest.

### Phase 7: Completion

When all tasks are `done`:

1. Orchestrator updates `state.json` with `phase: "complete"`.
2. Summarizes the run: tasks completed, bugs fixed, remaining risks.
3. Reports final status to the user.

## State Management

### Source of Truth Hierarchy

1. `.ai-control/state.json` — machine-readable, canonical
2. `.ai-control/task-board.md` — human-readable task status
3. `.ai-control/bug-board.md` — human-readable bug status
4. Chat history — supplementary only, not authoritative

### State Transitions

```
Task:  todo → in_progress → ready_for_test → done
                  │                   │
                  ▼                   ▼
               blocked          failed_test → in_progress (reassigned)

Bug:   open → in_fix → retest → closed
                         │
                         ▼
                      rejected
```

### Keeping State Aligned

The Orchestrator must ensure consistency between:
- `state.json` task entries
- `task-board.md` rows
- Individual task card `Status` fields
- Branch names and worktree paths

## Parallel Execution

### When to Parallelize

Tasks can run in parallel when:
- They have no dependencies on each other.
- They modify non-overlapping file paths.
- Any shared interfaces are defined in completed contract tasks.

### Parallel Topology Example

```
TASK-001 (DB schema)  ──┐
                         ├──→ TASK-004 (API integration)
TASK-002 (Auth module) ──┘
                               │
TASK-003 (UI scaffolding)      ▼
         │              TASK-005 (E2E tests)
         └──────────────────────┘
```

- **Parallel group 1**: TASK-001, TASK-002, TASK-003 (independent)
- **Serial after group 1**: TASK-004 (depends on TASK-001, TASK-002)
- **Serial after all**: TASK-005 (depends on TASK-003, TASK-004)

## Contract Tasks

When multiple executor tasks share an interface, schema, or API surface:

1. The Orchestrator creates a **contract task** that defines the shared artifact.
2. The contract task is marked as a dependency for all consuming tasks.
3. The contract executor implements only the shared definition (types, interfaces, schemas).
4. Consumer tasks are dispatched only after the contract task is `done`.

**Example:**

```
TASK-001: Define UserDTO schema (contract task)
TASK-002: Implement user API (depends on TASK-001)
TASK-003: Implement user UI (depends on TASK-001)
```

## Example Walkthrough

### Scenario: "Add a search feature to the product catalog"

**Step 1 — User invokes @Orchestrator:**
> "Add full-text search to the product catalog. Products should be searchable by name, description, and tags."

**Step 2 — Orchestrator creates PRFAQ and dispatches Planner:**
- Creates `.ai-control/state.json` with `run_id: "run-042"`.
- Creates `.ai-control/prfaq.md` with the search feature summary.
- Dispatches `runSubagent(agentName="Planner", ...)`.

**Step 3 — Planner returns task breakdown:**
```
TASK-001: Add search index schema (contract)
TASK-002: Implement search API endpoint (depends on TASK-001)
TASK-003: Add search UI component (depends on TASK-001)
TASK-004: Integration tests (depends on TASK-002, TASK-003)
```

**Step 4 — Orchestrator creates task cards and dispatches TASK-001:**
- Creates `.ai-control/tasks/TASK-001.md` through `TASK-004.md`.
- Dispatches Executor for TASK-001 (no dependencies).

**Step 5 — TASK-001 executor completes, Orchestrator dispatches TASK-002 and TASK-003 in parallel:**
- TASK-001 marked `done`.
- TASK-002 and TASK-003 dispatched simultaneously (both depend only on TASK-001).

**Step 6 — Both executors complete, Orchestrator dispatches testers:**
- Tester for TASK-002: runs API tests → passes.
- Tester for TASK-003: runs UI tests → finds a bug (search results not sorted).

**Step 7 — Orchestrator handles the bug:**
- Creates `BUG-001` (source: TASK-003, severity: medium).
- Reassigns to TASK-003's executor with updated acceptance criteria.
- Executor fixes, pushes. Tester retests → passes.

**Step 8 — TASK-004 dispatched (all dependencies satisfied):**
- Executor runs integration tests, all pass.
- Tester verifies. Orchestrator marks all tasks `done`.

**Step 9 — Orchestrator reports completion:**
> "Run run-042 complete. 4 tasks delivered, 1 bug found and fixed. Search feature is live."
