---
name: multi-agent-orchestrator
description: "Orchestrates model-aware Executor and Tester subagents for software delivery with persisted state in .ai-control/session.json. The Orchestrator handles planning, optional read-only research waves, model routing, and a complete audit log."
---

# Multi-Agent Orchestrator Skill

## Quick Start

1. Read `.ai-control/session.json` if it exists (single source of truth).
2. Read `.ai-control/CLAUDE.md` for project instructions.
3. Optionally run a read-only research wave for ambiguous or high-risk requests.
4. Plan tasks, assign complexity, execution profile, and selected model tiers, then write them inline into `session.json`.
5. Dispatch Executor and Tester subagents via `runSubagent` using self-contained prompt templates from `prompts.md`.
5. Persist all state changes, decisions, and audit log entries in `session.json`.

## Roles (3 roles)

| Role | Agent Name | Responsibility | May Write `session.json`? | May Write Business Code? |
|------|-----------|---------------|--------------------------|-------------------------|
| Orchestrator | `orchestrator` | Plan, route models, optionally run research waves, dispatch, arbitrate, maintain audit log | Yes (full) | No (unless workflow-control) |
| Executor | `executor` | Implement one task, verify, follow assigned execution profile and model context | Yes (own task only) | Yes |
| Tester | `tester` | Adversarial testing, code review. Strictly read-only | No | No |

## Subagent Dispatch

The Orchestrator auto-dispatches via `runSubagent` using self-contained prompt templates:

```
runSubagent(prompt="<filled research template from prompts.md>")
runSubagent(prompt="<filled executor template from prompts.md>")
runSubagent(prompt="<filled tester template from prompts.md>")
```

See [prompts.md](prompts.md) for ready-to-use templates. Each template embeds the full role definition — subagents do NOT auto-load `.github/agents/*.agent.md`.

## Session Schema

All state lives in `.ai-control/session.json`:

```json
{
  "run_id": "run-001",
  "goal": "<user request>",
  "model_policy": {},
  "orchestration_policy": {},
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
      "acceptance": ["criteria"],
      "verification": ["commands"],
      "model_override": {},
      "selected_models": {},
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
| `.ai-control/session.json` | Single source of truth |
| `.ai-control/CLAUDE.md` | Project instructions |
| `.ai-control/CLAUDE.local.md` | Local-only instructions |
| `.ai-control/templates/session.json` | Template for new sessions |

## Workflow Modes

| Mode | Condition | Behavior |
|------|-----------|----------|
| **Lightweight** | 1-3 tasks (default) | Sequential Executor → Tester per task |
| **Pipeline** | 4+ tasks or user requests | Parallel Executors by dependency graph → Tester per task |

Tester is **mandatory** in both modes.

Optional research waves are read-only planning aids, not a new permanent role.

## Routing Policy

- Orchestrator decides model tiers before dispatch.
- Routing priority: task override → complexity rule → role default → global fallback.
- Recommended tiers:
  - `flagship` — highest reasoning quality, e.g. `GPT-5.4`
  - `balanced` — standard implementation and analysis
  - `fast` — low-latency mechanical work
- Default role policy:
  - Orchestrator → `flagship`
  - Executor → `fast` for low, `balanced` for medium, `flagship` for high
  - Tester → `balanced` for low, `flagship` for medium and high

## Audit Log

The `log` array is the complete change history for cold-start recovery:

| Type | When | Key Fields |
|------|------|------------|
| `action` | Agent completes a step | `agent`, `action`, `detail` |
| `plan_change` | Requirements change | `trigger`, `before`, `after`, `tasks_added/modified/removed` |
| `iteration` | Fix→retest cycle | `iteration`, `tasks`, `issues_found`, `issues_fixed`, `detail` |

## Operating Rules

1. Orchestrator manages `log`, `decisions`, `phase`, `mode`, and task `issues` in `session.json`.
2. Executors may update their own task's `status`, `changed_paths`, `verified`, `notes`.
3. Testers are strictly read-only — they return reports to the Orchestrator.
4. All state lives in `session.json` — no separate task/bug/handoff files.
5. Do not treat chat history as authoritative when `session.json` has newer state.
6. Tester is mandatory for every task in every mode.
7. Research waves never write `session.json`; they only inform Orchestrator decisions.

## Main Agent Loop

1. **Read** `session.json` and `CLAUDE.md`.
2. **Research** optionally — run read-only research wave for ambiguous or high-risk work.
3. **Plan** tasks — decompose, assign complexity and model tiers, write into `session.json`, choose mode.
4. **Dispatch** Executor via `runSubagent` (sequential or parallel by dependency graph).
5. **Dispatch** Tester via `runSubagent` for every completed task.
6. **Arbitrate** Tester findings — record issues, re-dispatch Executor if needed (max 3 rounds, with model escalation when warranted).
6. **Record** all actions in `log` array.
7. **Repeat** until all tasks are `done` or user intervenes.

## Prompt Templates

Self-contained dispatch templates live in [prompts.md](prompts.md):

- Research prompt — read-only planning support, risk discovery, task-splitting hints
- Executor prompt — implement one task, verify, return result
- Tester prompt — adversarial testing, code review, return report

Each template embeds the full role definition — subagents do NOT auto-load `.github/agents/*.agent.md`.
