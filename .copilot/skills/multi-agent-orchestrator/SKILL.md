---
name: multi-agent-orchestrator
description: "Orchestrates Planner, Executor, and Tester subagents for parallel software delivery with persisted state under .ai-control/. Use when the user asks for multi-agent collaboration, parallel development, isolated executors, iterative bug fixing, or orchestrator-driven workflows."
---

# Multi-Agent Orchestrator Skill

## Quick Start

1. Read `.ai-control/state.json` if it exists.
2. If the workflow is new, create or refresh `prfaq.md`, `plan.md`, `task-board.md`, and task cards from templates in `.ai-control/templates/`.
3. Split work into isolated tasks with one branch and one worktree per executor task.
4. Dispatch Planner, Executor, and Tester subagents using `runSubagent` as needed.
5. Persist all workflow state changes under `.ai-control/`.
6. Do not treat chat memory as the source of truth when `.ai-control/` is available.

## Role Contracts

| Role | Agent Name | Responsibility | May Write `.ai-control/`? | May Write Business Code? |
|------|-----------|---------------|--------------------------|-------------------------|
| Orchestrator | `Orchestrator` | Understand request, maintain state, allocate tasks, dispatch subagents, replan after bugs | Yes | No (unless workflow-control) |
| Planner | `Planner` | Produce planning artifacts: task breakdown, dependencies, acceptance criteria | No | No |
| Executor | `Executor` | Implement exactly one task in one branch/worktree, verify, commit, push | No | Yes (scoped to allowed paths) |
| Tester | `Tester` | Test a target branch/commit, report evidence, draft bug cards | No | No |

## Subagent Dispatch

Use `runSubagent` with the agent names above:

```
runSubagent(agentName="Planner", prompt="<planner instructions>")
runSubagent(agentName="Executor", prompt="<executor instructions>")
runSubagent(agentName="Tester", prompt="<tester instructions>")
```

See [prompts.md](prompts.md) for ready-to-use prompt templates for each role.

## Canonical State Files

All workflow state lives under `.ai-control/`:

| File | Purpose |
|------|---------|
| `state.json` | Machine-readable source of truth |
| `prfaq.md` | Requirement summary (press-release / FAQ format) |
| `plan.md` | Current execution plan and parallel topology |
| `task-board.md` | Workflow board for task state |
| `bug-board.md` | Workflow board for bug state |
| `tasks/*.md` | Individual task cards |
| `bugs/*.md` | Individual bug cards |
| `handoffs/*.md` | Executor implementation handoffs |
| `reports/*.md` | Tester reports |

Templates for all of the above are in `.ai-control/templates/`.

## Operating Rules

1. **Only the Orchestrator** may write under `.ai-control/`.
2. Executors and Testers return structured output; they do not persist workflow state directly.
3. Prefer **contract-first tasks** when multiple executor tasks share interfaces or schema.
4. After any executor reports a successful push, mark the task `ready_for_test` and dispatch the Tester.
5. If the Tester reports a failure, create a bug card and reassign through the Orchestrator.
6. Keep task IDs, branch names, worktree paths, and verification commands **aligned across all files**.

## Main Agent Loop

1. **Read** state and current boards.
2. **Decide** whether the request needs planning, execution, testing, or replanning.
3. **Create or refresh** task cards before dispatching executors.
4. **Dispatch** only tasks whose dependencies are satisfied.
5. **Persist** all state transitions immediately after each subagent result.
6. **Repeat** until all tasks are `done` or the user intervenes.

## Prompt Templates

Use the templates in [prompts.md](prompts.md):

- Orchestrator prompt — main agent loop and state management
- Planner prompt — task decomposition and planning
- Executor prompt — scoped implementation with handoff
- Tester prompt — verification and evidence collection
- Bug re-assignment prompt — triage and reassignment
- Optional PUA injection block — pressure-driven behavior modifiers
