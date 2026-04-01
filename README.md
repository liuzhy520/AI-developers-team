# AI Developers Team — Multi-Agent Orchestrator for Copilot

A prompt engineering project that implements a **4-role multi-agent orchestration system** for VS Code GitHub Copilot. Four specialized AI agents collaborate through a context-aware workflow with persisted state, compacted memory, and resumable sessions.

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
│  • Maintains .ai-control/ session + context          │
│  • Dispatches subagents via runSubagent              │
│  • Compacts history, restores sessions               │
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
│  session.json │ context/ │ tasks/ │ bugs/            │
│  handoffs/    │ CLAUDE.md │ compacted.md             │
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
│   ├── CLAUDE.md                               # Project-level instructions
│   └── templates/                              # Minimal workflow schemas/templates
│       ├── session.json
│       ├── TASK-template.json
│       ├── BUG-template.json
│       └── HANDOFF-template.md
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
2. **Orchestrator** reads `.ai-control/session.json`, restores compacted context, and collects git context
3. **Orchestrator** decides whether the workflow is simple, standard, or complex
4. **Orchestrator** dispatches **Planner** when decomposition is needed
5. **Orchestrator** creates JSON task cards from planner output when needed
6. **Orchestrator** dispatches **Executor(s)** with injected workflow context
7. **Executor** implements in isolated branch/worktree, returns structured JSON handoff + context update
8. **Orchestrator** marks task `ready_for_test`, dispatches **Tester**
9. **Tester** verifies and reports structured evidence
10. If passed → **Orchestrator** marks `done`
11. If failed → **Orchestrator** creates a JSON bug card and reassigns
12. As sessions grow, **Orchestrator** compacts older context into `.ai-control/context/compacted.md`

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
    - Create or refresh `.ai-control/session.json`
    - Create `.ai-control/CLAUDE.md` if needed
    - Decompose work into JSON task cards when needed
    - Dispatch subagents with workflow context
    - Track progress in session state

### Resuming an Existing Workflow

1. Open Copilot Chat and select **@Orchestrator**
2. Say "resume" or describe the next action
3. The Orchestrator reads `.ai-control/session.json` and `.ai-control/context/compacted.md` and picks up where it left off

## Prompt Templates

All prompt templates are in [.copilot/skills/multi-agent-orchestrator/prompts.md](.copilot/skills/multi-agent-orchestrator/prompts.md):

- **Orchestrator prompt** — main agent loop
- **Planner prompt** — task decomposition
- **Executor prompt** — scoped implementation
- **Tester prompt** — verification and reporting
- **Bug re-assignment prompt** — triage and reassignment
- **PUA injection block** — optional high-pressure behavior modifier

## Design Principles

1. **Persisted session state over chat memory** — `.ai-control/session.json` is the source of truth
2. **Task isolation** — each executor works in a dedicated branch/worktree with scoped paths
3. **Contract-first** — shared interfaces get their own task before consumers
4. **Evidence-based completion** — tasks are done only when verification passes
5. **Structured handoffs** — executors and testers return machine-parseable output
6. **Compacted memory** — older context is summarized instead of lost
7. **Resumable orchestration** — sessions can continue across conversations