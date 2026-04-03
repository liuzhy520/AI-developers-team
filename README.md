# AI Developers Team — Multi-Agent Orchestrator for Copilot

**Version:** v1.0.0
**Milestone:** Ceremony-Minimized Protocol Release (2026-04-03)

A prompt engineering project that implements a **3-role multi-agent orchestration system** for VS Code GitHub Copilot. Three specialized AI agents collaborate through an automated workflow with persisted state, adversarial testing, and a complete audit trail.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   User Request                       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│                 ORCHESTRATOR                          │
│  • Plans tasks (absorbs Planner role)                │
│  • Auto-dispatches Executor & Tester via runSubagent │
│  • Arbitrates Tester findings                        │
│  • Maintains audit log in session.json               │
└───────────────┬──────────────┬──────────────────────┘
                │              │
                ▼              ▼
         ┌──────────┐  ┌──────────┐
         │ EXECUTOR │  │  TESTER  │
         │          │  │          │
         │Implements│  │Adversarial│
         │one task, │  │testing,  │
         │verifies, │  │code      │
         │updates   │  │review,   │
         │session   │  │read-only │
         └──────────┘  └──────────┘
                │              │
                ▼              ▼
┌─────────────────────────────────────────────────────┐
│           .ai-control/session.json                   │
│    Single source of truth — tasks, log, decisions    │
└─────────────────────────────────────────────────────┘
```

## Roles

| Role | Agent File | Responsibility |
|------|-----------|---------------|
| **Orchestrator** | `.github/agents/orchestrator.agent.md` | Plan, dispatch, arbitrate, audit log |
| **Executor** | `.github/agents/executor.agent.md` | Implement one task, verify, update own status |
| **Tester** | `.github/agents/tester.agent.md` | Adversarial testing, code review, strictly read-only |

## Project Structure

```
.
├── .github/
│   ├── copilot-instructions.md                  # Base protocol (3 roles, 2 modes)
│   ├── instructions/
│   │   ├── task-isolation.instructions.md       # Task isolation rules
│   │   └── handoff-and-test.instructions.md     # Executor/Tester contract
│   └── agents/
│       ├── orchestrator.agent.md                # Orchestrator (plan + dispatch + arbitrate)
│       ├── executor.agent.md                    # Executor (implement + verify)
│       └── tester.agent.md                      # Tester (adversarial, read-only)
├── .copilot/
│   └── skills/
│       └── multi-agent-orchestrator/
│           ├── SKILL.md                         # Skill definition
│           └── prompts.md                       # Self-contained dispatch templates
├── .ai-control/
│   ├── CLAUDE.md                               # Project-level instructions
│   └── templates/
│       └── session.json                        # Session template
└── docs/
    └── workflow-guide.md                        # Detailed workflow walkthrough
```

## How It Works

### Copilot Integration Points

| Copilot Feature | File Location | Purpose |
|----------------|--------------|---------|
| **Custom Instructions** | `.github/copilot-instructions.md` | Global rules applied to every conversation |
| **Scoped Instructions** | `.github/instructions/*.instructions.md` | Context-specific rules triggered by file patterns |
| **Custom Agents** | `.github/agents/*.agent.md` | Named agent modes selectable in Copilot Chat |
| **Skills** | `.copilot/skills/*/SKILL.md` | On-demand capabilities invoked by agents |
| **Subagent Dispatch** | `runSubagent(prompt=...)` | Automatic dispatch with self-contained prompts |

### Workflow Lifecycle

1. **User** selects **@Orchestrator** and describes a feature or change
2. **Orchestrator** reads `.ai-control/session.json` if resuming
3. **Orchestrator** plans tasks, writes them inline into `session.json`
4. **Orchestrator** auto-dispatches **Executor** via `runSubagent` with filled prompt template
5. **Executor** implements, verifies, updates own task status in `session.json`
6. **Orchestrator** auto-dispatches **Tester** via `runSubagent` (mandatory for every task)
7. **Tester** performs adversarial testing, code review, returns structured report
8. If issues found → **Orchestrator** records issues, re-dispatches Executor → Tester (max 3 rounds)
9. When all tasks done → **Orchestrator** marks `phase: "complete"`

### Workflow Modes

| Mode | When | Behavior |
|------|------|----------|
| **Lightweight** | 1-3 tasks (default) | Sequential Executor → Tester per task |
| **Pipeline** | 4+ tasks or user requests | Parallel Executors by dependency graph → Tester per task |

### Audit Log

The `log` array in `session.json` is the complete change history. A new AI reading only `goal + tasks + log` can reconstruct full context.

## Quick Start

### Starting a Fresh Workflow

1. Open Copilot Chat, select **@Orchestrator**
2. Describe your feature or change request
3. The Orchestrator will plan, dispatch, test, and track everything automatically

### Resuming an Existing Workflow

1. Open Copilot Chat, select **@Orchestrator**
2. Say "resume" — the Orchestrator reads `session.json` and continues

## Design Principles

1. **Single source of truth** — `session.json` holds all tasks, issues, and audit log
2. **Adversarial testing** — Tester finds bugs, not rubber-stamps
3. **Automatic dispatch** — Orchestrator drives the full loop via `runSubagent`
4. **Minimal ceremony** — no separate task cards, bug files, or handoff files
5. **Cold-start recovery** — `goal + tasks + log` is sufficient for a new AI to continue
6. **Mandatory testing** — Tester runs for every task in every mode