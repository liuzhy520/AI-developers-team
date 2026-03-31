# Multi-Agent Base Protocol

This file is the canonical base protocol for orchestrated multi-agent delivery with persisted state. It applies to every Copilot conversation in this workspace.

## Core Rules

- Treat `.ai-control/state.json` as the canonical workflow state.
- Read `.ai-control/state.json` before substantial work if it exists.
- Only the orchestrator agent may create or update files under `.ai-control/`.
- Planner only plans. Tester only tests. Executors only implement assigned tasks.
- Every task must include `id`, `title`, `owner`, `branch`, `worktree`, `allowed_paths`, `depends_on`, `acceptance`, and `verification`.
- Every bug must include `id`, `source_task`, `owner`, `severity`, `repro`, `actual`, and `expected`.
- Use one branch and one worktree per executor task.
- Do not rely on chat history as the source of truth when `.ai-control/` has newer state.

## Role Summary

| Role | Responsibility | May Write `.ai-control/`? | May Write Business Code? |
|------|---------------|--------------------------|-------------------------|
| Orchestrator | Understand request, maintain state, allocate tasks, dispatch subagents, replan after bugs | Yes | No (unless workflow-control task) |
| Planner | Produce planning artifacts: task breakdown, dependencies, acceptance criteria | No | No |
| Executor | Implement exactly one task in one branch/worktree, verify, commit, push | No | Yes (scoped to allowed paths) |
| Tester | Test a target branch/commit, report evidence, draft bug cards | No | No |

## Workflow State Files

All workflow state lives under `.ai-control/`:

- `state.json` — machine-readable source of truth
- `prfaq.md` — requirement summary (press-release / FAQ format)
- `plan.md` — current execution plan and parallel topology
- `task-board.md` — workflow board for task state tracking
- `bug-board.md` — workflow board for bug state tracking
- `tasks/*.md` — individual task cards
- `bugs/*.md` — individual bug cards
- `handoffs/*.md` — executor implementation handoffs
- `reports/*.md` — tester reports

## Subagent Dispatch

Use `runSubagent` with the following agent names for multi-agent orchestration:

- `Orchestrator` — main agent loop, state management, dispatch
- `Planner` — read-only planning and task decomposition
- `Executor` — scoped implementation of a single task
- `Tester` — read-only testing and evidence collection

## Templates

Workflow templates are stored in `.ai-control/templates/`. Copy them to the appropriate location when creating new task cards, bug cards, handoffs, or reports.
