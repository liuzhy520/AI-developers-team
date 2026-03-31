# .ai-control — Workflow State Directory

This directory holds the canonical workflow state for multi-agent orchestrated delivery. It is managed exclusively by the **Orchestrator** agent.

## Directory Structure

```
.ai-control/
├── README.md              ← this file
├── state.json             ← machine-readable source of truth (runtime)
├── prfaq.md               ← requirement summary (runtime)
├── plan.md                ← current execution plan (runtime)
├── task-board.md          ← task workflow board (runtime)
├── bug-board.md           ← bug workflow board (runtime)
├── tasks/                 ← individual task cards (runtime)
├── bugs/                  ← individual bug cards (runtime)
├── handoffs/              ← executor handoff records (runtime)
├── reports/               ← tester report records (runtime)
└── templates/             ← reusable templates (checked in)
    ├── state.json
    ├── prfaq.md
    ├── plan.md
    ├── task-board.md
    ├── bug-board.md
    ├── TASK-template.md
    ├── BUG-template.md
    ├── HANDOFF-template.md
    └── TEST-REPORT-template.md
```

## Rules

1. **Only the Orchestrator** may create or modify files in this directory.
2. Subagents (Planner, Executor, Tester) return structured output — the Orchestrator persists it.
3. `state.json` is the machine-readable source of truth; markdown boards are human-readable views.
4. Templates live in `templates/` — copy them to the parent directories when creating new items.
5. Chat history is **not** the source of truth when `.ai-control/` has newer state.

## Usage

### Starting a New Workflow

1. Copy `templates/state.json` to `./state.json` and fill in the `run_id` and `goal`.
2. Copy `templates/prfaq.md` to `./prfaq.md` and draft the requirement summary.
3. Copy `templates/plan.md` to `./plan.md` and define the execution plan.
4. Copy `templates/task-board.md` to `./task-board.md`.
5. Copy `templates/bug-board.md` to `./bug-board.md`.
6. For each task, copy `templates/TASK-template.md` to `tasks/TASK-NNN.md`.

### During Execution

- After an executor hands off, copy `templates/HANDOFF-template.md` to `handoffs/HANDOFF-TASK-NNN.md`.
- After a tester reports, copy `templates/TEST-REPORT-template.md` to `reports/REPORT-TASK-NNN.md`.
- If a bug is found, copy `templates/BUG-template.md` to `bugs/BUG-NNN.md`.
- Update `state.json`, `task-board.md`, and `bug-board.md` after every state transition.
