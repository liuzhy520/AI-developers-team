# .ai-control — Workflow State Directory

This directory holds the canonical workflow state for multi-agent orchestrated delivery.

## Directory Structure

```
.ai-control/
├── README.md                  ← this file
├── session.json               ← single source of truth (tasks, log, decisions)
├── CLAUDE.md                  ← project-level instructions
├── CLAUDE.local.md            ← local-only instructions (optional, untracked)
└── templates/
    └── session.json           ← template for new sessions
```

## Rules

1. `session.json` is the **single source of truth** — all tasks, issues, and audit log live here.
2. The **Orchestrator** manages `log`, `decisions`, `phase`, `mode`, and task `issues`.
3. **Executors** may update their own task's `status`, `changed_paths`, `verified`, `notes`.
4. **Testers** are strictly read-only — they return reports to the Orchestrator.
5. Chat history is **not** authoritative when `session.json` has newer state.
6. A new AI reading `goal + model_policy + orchestration_policy + tasks + log` can reconstruct full context (cold-start).
7. `model_policy` and `orchestration_policy` define how tasks are routed, escalated, and optionally researched before execution.
8. Task routing metadata such as `complexity`, `selected_models`, `execution_profile`, and `dispatch_strategy` belong in `session.json`, not in side files.

## Usage

### Starting a New Workflow

1. Create `session.json` from `templates/session.json`.
2. Create or update `CLAUDE.md` with project commands, architecture, and conventions.

### During Execution

- Orchestrator updates `session.json` after every dispatch, test, and arbitration.
- All changes are tracked in the `log` array.
- No separate task card, bug card, or handoff files are needed.
- Optional read-only research waves inform planning, but only the Orchestrator persists the outcome into `session.json`.
