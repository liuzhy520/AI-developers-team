# AI Developers Team — Multi-Agent Orchestrator for Copilot

A prompt engineering project that implements a **4-role multi-agent orchestration system** for VS Code GitHub Copilot. Four specialized AI agents collaborate through a structured workflow with persisted state to deliver software in parallel.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   User Request                       │
└──────────────────────┬──────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│                 ORCHESTRATOR                          │
│  • Understands request                               │
│  • Maintains .ai-control/ state                      │
│  • Dispatches subagents via runSubagent              │
│  • Persists results, replans after bugs              │
└───┬──────────────┬──────────────┬───────────────────┘
    │              │              │
    ▼              ▼              ▼
┌────────┐  ┌──────────┐  ┌──────────┐
│PLANNER │  │ EXECUTOR │  │  TESTER  │
│        │  │          │  │          │
│Plans   │  │Implements│  │Tests &   │
│tasks,  │  │one task  │  │reports   │
│deps,   │  │per branch│  │evidence, │
│criteria│  │& worktree│  │drafts    │
│        │  │          │  │bug cards │
└────────┘  └──────────┘  └──────────┘
    │              │              │
    ▼              ▼              ▼
┌─────────────────────────────────────────────────────┐
│              .ai-control/ (Persisted State)           │
│  state.json │ task-board.md │ bug-board.md           │
│  tasks/     │ handoffs/     │ reports/    │ bugs/    │
└─────────────────────────────────────────────────────┘
```

## Roles

| Role | Agent File | Responsibility |
|------|-----------|---------------|
| **Orchestrator** | `.github/agents/orchestrator.agent.md` | State management, task allocation, subagent dispatch, replanning |
| **Planner** | `.github/agents/planner.agent.md` | Task decomposition, dependency analysis, acceptance criteria |
| **Executor** | `.github/agents/executor.agent.md` | Scoped implementation in isolated branch/worktree |
| **Tester** | `.github/agents/tester.agent.md` | Verification, evidence collection, bug card drafting |

## Project Structure

```
.
├── .github/
│   ├── copilot-instructions.md                  # Global base protocol (always active)
│   ├── instructions/
│   │   ├── task-isolation.instructions.md       # Task isolation rules
│   │   └── handoff-and-test.instructions.md     # Handoff/test contract
│   └── agents/
│       ├── orchestrator.agent.md                # Orchestrator agent definition
│       ├── planner.agent.md                     # Planner agent definition
│       ├── executor.agent.md                    # Executor agent definition
│       └── tester.agent.md                      # Tester agent definition
├── .copilot/
│   └── skills/
│       └── multi-agent-orchestrator/
│           ├── SKILL.md                         # Orchestrator skill definition
│           └── prompts.md                       # All prompt templates
├── .ai-control/
│   ├── README.md                                # Usage guide
│   └── templates/                               # Reusable workflow templates
│       ├── state.json
│       ├── prfaq.md
│       ├── plan.md
│       ├── task-board.md
│       ├── bug-board.md
│       ├── TASK-template.md
│       ├── BUG-template.md
│       ├── HANDOFF-template.md
│       └── TEST-REPORT-template.md
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
| **Subagent Dispatch** | `runSubagent(agentName, prompt)` | Programmatic dispatch of named agents |

### Workflow Lifecycle

1. **User** describes a feature or change request
2. **Orchestrator** reads `.ai-control/state.json`, creates PRFAQ and plan
3. **Orchestrator** dispatches **Planner** for task decomposition
4. **Orchestrator** creates task cards from planner output
5. **Orchestrator** dispatches **Executor(s)** for parallel implementation
6. **Executor** implements in isolated branch/worktree, returns handoff
7. **Orchestrator** marks task `ready_for_test`, dispatches **Tester**
8. **Tester** verifies and reports evidence
9. If passed → **Orchestrator** marks `done`
10. If failed → **Orchestrator** creates bug card, reassigns

## Quick Start

### Using Agent Modes

In VS Code Copilot Chat, select an agent mode:

- **@Orchestrator** — Start or resume a multi-agent workflow
- **@Planner** — Get a task breakdown and plan
- **@Executor** — Implement a specific task card
- **@Tester** — Test and verify a task

### Starting a Fresh Workflow

1. Open Copilot Chat and select **@Orchestrator**
2. Describe your feature or change request
3. The Orchestrator will:
   - Create `.ai-control/state.json` and workflow boards
   - Decompose work into task cards
   - Dispatch subagents as needed
   - Track progress on the task board

### Resuming an Existing Workflow

1. Open Copilot Chat and select **@Orchestrator**
2. Say "resume" or describe the next action
3. The Orchestrator reads `.ai-control/state.json` and picks up where it left off

## Prompt Templates

All prompt templates are in [.copilot/skills/multi-agent-orchestrator/prompts.md](.copilot/skills/multi-agent-orchestrator/prompts.md):

- **Orchestrator prompt** — main agent loop
- **Planner prompt** — task decomposition
- **Executor prompt** — scoped implementation
- **Tester prompt** — verification and reporting
- **Bug re-assignment prompt** — triage and reassignment
- **PUA injection block** — optional high-pressure behavior modifier

## Design Principles

1. **Persisted state over chat memory** — `.ai-control/` is the source of truth
2. **Task isolation** — each executor works in a dedicated branch/worktree with scoped paths
3. **Contract-first** — shared interfaces get their own task before consumers
4. **Evidence-based completion** — tasks are done only when verification passes
5. **Structured handoffs** — executors and testers return machine-parseable output