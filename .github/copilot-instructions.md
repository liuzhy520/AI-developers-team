# Multi-Agent Base Protocol

This file is the canonical base protocol for orchestrated multi-agent delivery. It applies to every Copilot conversation in this workspace.

## Core Rules

- Treat `.ai-control/session.json` as the canonical workflow state — the single source of truth.
- Read `.ai-control/session.json` before substantial work if it exists.
- Do not rely on chat history when `session.json` has newer state.
- All tasks, issues, and audit logs live inline in `session.json`. No separate task/bug/handoff files.
- Keep the main workflow as a 3-role delivery pipeline. Do not introduce free-form teammate collaboration as the default mode.
- The Orchestrator is the only agent that selects subagent model tiers, execution profiles, and dispatch strategies.
- Tester remains mandatory for every task, even when planning includes optional read-only research waves.

## Roles (3 roles)

| Role | Responsibility | May Write `session.json`? | May Write Business Code? |
|------|---------------|--------------------------|-------------------------|
| Orchestrator | Plan tasks, optionally run read-only research waves, select models, auto-dispatch Executor/Tester, arbitrate bugs, maintain audit log | Yes (full) | No (unless workflow-control) |
| Executor | Implement exactly one task, run verification, follow assigned execution profile and model context | Yes (own task only: `status`, `changed_paths`, `verified`, `notes`) | Yes |
| Tester | Adversarial testing — find bugs Executor missed. Strictly read-only | No | No |

## Subagent Dispatch

Use `runSubagent` with these agent names:

- `orchestrator` — main loop (plan + dispatch + arbitrate)
- `executor` — scoped implementation of a single task
- `tester` — adversarial testing and reporting

Self-contained prompt templates live in `.copilot/skills/multi-agent-orchestrator/prompts.md`. These templates embed the full role definition — subagents do NOT auto-load `.github/agents/*.agent.md`.

The Orchestrator may also run an optional read-only research wave before task execution for high-ambiguity work. Research results inform planning but do not update workflow state directly.

## Workflow State

All state lives in `.ai-control/session.json`:

```json
{
  "run_id": "run-001",
  "goal": "<user request>",
  "model_policy": {
    "tiers": {"flagship": "GPT-5.4", "balanced": "", "fast": ""},
    "role_defaults": {"orchestrator": "flagship", "executor": "balanced", "tester": "flagship"}
  },
  "orchestration_policy": {
    "research_wave": {"enabled": true, "max_parallel_agents": 3, "write_state": false}
  },
  "mode": "lightweight | pipeline",
  "phase": "planning | executing | testing | complete",
  "updated_at": "ISO",
  "decisions": [],
  "tasks": [
    {
      "id": "TASK-001",
      "title": "...",
      "status": "todo | doing | testing | done | blocked",
      "depends_on": [],
      "complexity": "low | medium | high",
      "acceptance": ["human-readable criteria"],
      "verification": ["pytest tests/ -v"],
      "model_override": {},
      "selected_models": {
        "executor": {"tier": "balanced", "name": "", "reason": ""},
        "tester": {"tier": "flagship", "name": "", "reason": ""}
      },
      "model_history": [],
      "execution_profile": "read-heavy | write-heavy | adversarial-test | parallel-safe",
      "dispatch_strategy": "sequential | parallel_wave | retest_loop",
      "changed_paths": [],
      "verified": [],
      "issues": [],
      "notes": ""
    }
  ],
  "log": []
}
```

## Supporting Files

| File | Purpose |
|------|---------|
| `.ai-control/session.json` | Single source of truth — tasks, log, decisions |
| `.ai-control/CLAUDE.md` | Project-level instructions (build commands, conventions) |
| `.ai-control/CLAUDE.local.md` | Local-only instructions (not committed) |
| `.ai-control/templates/session.json` | Template for new sessions |

## Workflow Modes

| Mode | Condition | Behavior |
|------|-----------|----------|
| **Lightweight** | 1-3 tasks (default) | Orchestrator plans → sequential Executor → Tester per task |
| **Pipeline** | 4+ tasks or user requests | Orchestrator plans → parallel Executors (by dependency graph) → Tester per task |

Tester is **mandatory** in both modes. No exceptions.

For ambiguous or high-risk work, the Orchestrator may insert a read-only research wave before the main execution loop. This is a planning aid, not a fourth permanent role.

## Model Routing

- Model routing is a protocol-level concern. The Orchestrator decides model tiers before dispatch.
- Routing priority is: task override → complexity rule → role default → global fallback.
- Recommended mixed-catalog tiers:
  - `flagship` — highest reasoning quality, e.g. `GPT-5.4`
  - `balanced` — general implementation and analysis
  - `fast` — low-latency, low-cost local or mechanical work
- Default role policy:
  - Orchestrator → `flagship`
  - Executor → `fast` for low complexity, `balanced` for medium, `flagship` for high
  - Tester → `balanced` for low complexity, `flagship` for medium and high complexity
- Escalate the Executor one tier when a task is blocked, enters a second fix→retest loop, or carries high-severity findings.

## Execution Profiles

- Each task records an `execution_profile` chosen by the Orchestrator.
- Supported profiles are:
  - `read-heavy` — research or code understanding with no intended edits
  - `write-heavy` — normal implementation work
  - `adversarial-test` — hostile validation and code review
  - `parallel-safe` — independent work suitable for a pipeline wave
- Execution profiles guide prompts and review scope. They do not create a second source of truth outside `session.json`.

## Audit Log

The `log` array in `session.json` is the complete change history. A new AI cold-starting reads `goal + model_policy + orchestration_policy + tasks + log` to reconstruct full context.

| Type | When | Key Fields |
|------|------|------------|
| `action` | Agent completes a step | `agent`, `action`, `detail` |
| `plan_change` | Requirements change | `trigger`, `before`, `after`, `tasks_added/modified/removed` |
| `iteration` | Fix→retest cycle | `iteration`, `tasks`, `issues_found`, `issues_fixed`, `detail` |

Include model-routing and dispatch rationale in `detail`, for example:
- `selected executor=balanced, tester=flagship for TASK-002 due to medium complexity`
- `spawned read-only research wave for auth, data, and API surfaces`
