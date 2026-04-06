---
applyTo: ".ai-control/**"
---

# Task Isolation

These rules apply when working with tasks in `session.json`.

## Executor Constraints

- Executors implement one task at a time.
- Executors may only update their own task's fields in `session.json`: `status`, `changed_paths`, `verified`, `notes`.
- Executors do not update other tasks, the `log` array, `decisions`, or `phase`.
- If a task requires changes that conflict with another in-progress task, stop and report `"blocked"`.
- Do not widen scope — record follow-ups in `notes` instead of implementing them.
- Executors must respect task routing metadata chosen by the Orchestrator: `complexity`, `selected_models`, `execution_profile`, and `dispatch_strategy`.

## Task Schema

Tasks live inline in `session.json`. Each task has:

- `id` — unique identifier (e.g. `TASK-001`)
- `title` — short description
- `status` — `todo | doing | testing | done | blocked`
- `depends_on` — prerequisite task IDs
- `complexity` — `low | medium | high`
- `acceptance` — acceptance criteria as an array of strings
- `verification` — commands to run before claiming completion
- `model_override` — optional task-specific routing override
- `selected_models` — latest Executor/Tester routing choice and rationale
- `model_history` — prior routing choices and escalation notes
- `execution_profile` — `read-heavy | write-heavy | adversarial-test | parallel-safe`
- `dispatch_strategy` — `sequential | parallel_wave | retest_loop`
- `changed_paths` — files modified (filled by Executor)
- `verified` — verification results (filled by Executor and Tester)
- `issues` — problems found by Tester (filled by Orchestrator)
- `notes` — observations, risks, blockers

## Dependencies

- Tasks with `depends_on` entries must wait until all dependencies are `"done"`.
- In pipeline mode, the Orchestrator dispatches independent tasks in parallel.
- If a dependency is `"blocked"`, dependent tasks are also blocked.
- Optional read-only research waves may run before task execution, but research subagents do not claim tasks and do not write `session.json`.

## State Transitions

```
todo → doing → testing → done
         │         │
         ▼         ▼
      blocked    doing (fix issues → retest)
```
