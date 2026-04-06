# Workflow Guide

A detailed walkthrough of the upgraded 3-role multi-agent orchestration workflow for VS Code GitHub Copilot.

## Overview

```
User Request → Orchestrator (research/plan/route) → Executor(s) → Tester(s) → Done
                    ↑                                           │
                    └────────────── Fix→Retest Loop ←───────────┘
```

Three specialized roles:

- **Orchestrator**: Plans tasks, optionally runs read-only research waves, assigns model tiers and execution profiles, auto-dispatches Executor and Tester via `runSubagent`, arbitrates bugs, maintains audit log.
- **Executor**: Implements one task, runs verification, updates own task status in `session.json`, follows assigned routing context.
- **Tester**: Adversarial testing — finds bugs Executor missed. Strictly read-only. Mandatory in every workflow.

All state lives in a single file: `.ai-control/session.json`.

## Prerequisites

1. VS Code with GitHub Copilot extension.
2. This repository cloned locally.
3. Agent modes visible in Copilot Chat (`.github/agents/*.agent.md`).

## Workflow Modes

| Mode | Condition | Execution Pattern |
|------|-----------|-------------------|
| **Lightweight** | 1-3 tasks (default) | Sequential: one Executor → one Tester per task |
| **Pipeline** | 4+ tasks or user requests | Parallel: independent Executors dispatched simultaneously → Tester per task |

Tester is **mandatory** in both modes. No exceptions.

## Routing Layer

The upgraded workflow adds a routing layer controlled by the Orchestrator:

- `complexity` — `low | medium | high`
- `execution_profile` — `read-heavy | write-heavy | adversarial-test | parallel-safe`
- `selected_models` — Executor and Tester model tier/name plus rationale
- `dispatch_strategy` — `sequential | parallel_wave | retest_loop`

Recommended mixed-catalog tiers:

- `flagship` — highest reasoning quality, e.g. `GPT-5.4`
- `balanced` — normal implementation and analysis work
- `fast` — low-latency mechanical or localized tasks

Default routing policy:

- Orchestrator → `flagship`
- Executor → `fast` for low, `balanced` for medium, `flagship` for high
- Tester → `balanced` for low, `flagship` for medium and high

If a task becomes blocked, enters a second fix→retest loop, or carries high-severity findings, the Orchestrator escalates the Executor one tier and records that in `model_history` and `log`.

## Phases

### Phase 1: Planning

The Orchestrator handles planning directly (no separate Planner dispatch).

For ambiguous or high-risk requests, planning may start with a **read-only research wave**:

1. Orchestrator spawns up to 3 read-only research subagents.
2. Each explores one scope, such as architecture, risk, or code-path discovery.
3. Research results feed planning, model routing, and test recommendations.
4. Research subagents do not edit files or write `session.json`.

**Starting fresh:**
1. User opens Copilot Chat, selects **@Orchestrator**.
2. User describes the request.
3. Orchestrator reads `.ai-control/CLAUDE.md` for project instructions.
4. Orchestrator optionally runs research wave if needed.
5. Orchestrator decomposes the request into tasks with acceptance criteria and verification commands.
6. Orchestrator assigns each task a complexity, execution profile, dispatch strategy, and selected model tiers.
7. Orchestrator writes tasks into `session.json`, chooses lightweight or pipeline mode.
6. Orchestrator records plan in the `log` array.

**Resuming:**
1. Orchestrator reads `session.json` — the `goal + model_policy + orchestration_policy + tasks + log` contain full context.
2. Continues from current phase without asking user to repeat.

### Phase 2: Execution

Orchestrator dispatches Executor subagents using self-contained prompt templates from `prompts.md`:

```
runSubagent(prompt="<filled executor template>")
```

- **Lightweight**: One Executor at a time. Wait for result before dispatching next.
- **Pipeline**: Analyze dependency graph. Dispatch all independent Executors in parallel. Next wave after current wave completes.

After each Executor returns:
1. Orchestrator verifies the task status update in `session.json`.
2. Verifies that routing fields remain intact and adds any escalation notes to `model_history` if applicable.
3. Records an `action` log entry.
3. Proceeds to testing.

### Phase 3: Adversarial Testing

For EVERY completed task, Orchestrator dispatches the Tester:

```
runSubagent(prompt="<filled tester template>")
```

The Tester receives: acceptance criteria, verification commands, changed paths, code diff summary, Executor's verification results, and planning test recommendations.
The Tester also receives task complexity, execution profile, selected tester model, and any escalation reason.

**Tester requirements:**
- Must read the actual code changes (not just run commands).
- Must design independent test scenarios beyond Executor's verification.
- Must test adversarial inputs, failure modes, boundary conditions.
- Must NOT modify any files — returns a structured JSON report.
- Must NOT report "all passed" without documenting independent testing evidence.

### Phase 4: Arbitration & Fix Loop

When the Tester finds issues:

1. Orchestrator reviews the Tester's report.
2. Records issues into the task's `issues` array in `session.json`.
3. Records an `iteration` log entry.
4. If needed, escalates the Executor model tier and appends a `model_history` entry.
5. Re-dispatches Executor with the Tester's findings to fix the issues.
6. Re-dispatches Tester to retest after fix.
6. **Max 3 rounds** per task. After 3 rounds → reports to user for decision.

### Phase 5: Completion

When all tasks are `"done"`:
1. Orchestrator sets `phase: "complete"` in `session.json`.
2. Records final log entry.
3. Reports summary to user.

## Requirement Changes

When the user changes requirements mid-workflow:

1. Orchestrator records a `plan_change` log entry with before/after summaries.
2. Updates/adds/removes tasks in `session.json`.
3. Re-dispatches affected Executors and Testers.

## State Management

### Single Source of Truth

`.ai-control/session.json` contains everything:
- `goal` — what we're building
- `model_policy` — role defaults, tiers, escalation rules
- `orchestration_policy` — research-wave and routing preferences
- `mode` — lightweight or pipeline
- `phase` — current workflow phase
- `decisions` — key decisions made
- `tasks` — all tasks with status, complexity, routing metadata, acceptance, verification, issues
- `log` — complete audit trail

A new AI cold-starting reads `goal + model_policy + orchestration_policy + tasks + log` to reconstruct full context.

### Task State Transitions

```
todo → doing → testing → done
         │         │
         ▼         ▼
      blocked    doing (fix→retest loop)
```

### Audit Log Types

| Type | When | Key Fields |
|------|------|------------|
| `action` | Agent completes a step | `agent`, `action`, `detail` |
| `plan_change` | Requirements change | `trigger`, `before`, `after`, `tasks_added/modified/removed` |
| `iteration` | One fix→retest cycle | `iteration`, `tasks`, `issues_found`, `issues_fixed`, `detail` |

## Example Walkthrough

### Scenario: "Add search to product catalog"

**Step 1 — User invokes @Orchestrator:**
> "Add full-text search. Products searchable by name, description, and tags."

**Step 2 — Orchestrator plans and routes:**
- Creates `session.json` with `run_id: "run-042"`, mode: `"pipeline"`.
- Runs a read-only research wave for data model and search surface because the request spans indexing and ranking.
- Decomposes into 4 tasks inline in `session.json`.
- Assigns `complexity`, `execution_profile`, and `selected_models` for each task.
- Records plan and routing rationale in `log`.

**Step 3 — Orchestrator dispatches TASK-001 (no dependencies):**
```
runSubagent(prompt="<executor template with TASK-001 context>")
```
Executor implements, updates task status to `"done"`.

**Step 4 — Orchestrator dispatches Tester for TASK-001:**
```
runSubagent(prompt="<tester template with TASK-001 context>")
```
Tester reviews code, runs adversarial tests, finds no issues → `"passed"`.

**Step 5 — TASK-002 and TASK-003 dispatched in parallel (both depend only on TASK-001):**
Both Executors complete. Orchestrator dispatches Testers for each, keeping TASK-003 on `flagship` review because it touches ranking logic.

**Step 6 — Tester for TASK-003 finds a bug:**
```json
{"severity": "medium", "description": "Search results not sorted by relevance", "evidence": "..."}
```

**Step 7 — Orchestrator arbitrates:**
- Records issue in TASK-003's `issues` array.
- Records `iteration` log entry.
- Escalates TASK-003 Executor from `balanced` to `flagship` because the issue affects ranking correctness.
- Re-dispatches Executor for TASK-003 with the bug context.
- After fix, re-dispatches Tester → passes.

**Step 8 — TASK-004 dispatched (all dependencies met), tested, passes.**

**Step 9 — Orchestrator reports:**
> "Run run-042 complete. 4 tasks delivered, 1 bug found and fixed."
