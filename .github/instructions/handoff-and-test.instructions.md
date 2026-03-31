---
applyTo: ".ai-control/handoffs/**,.ai-control/reports/**"
---

# Handoff and Test Contract

These rules apply when working with executor handoffs and tester reports.

## Executor Handoff Format

Executors must return structured output containing:

- `task_id` — the task being handed off
- `status` — `success` | `blocked` | `failed`
- `branch` — the branch containing the implementation
- `commit_sha` — the final commit hash
- `changed_paths` — list of files modified
- `verification` — verification command output or summary
- `open_risks` — any risks or concerns discovered during implementation
- `handoff_summary` — short summary for the orchestrator to persist

## Tester Report Format

Testers must return structured output containing:

- `task_id` — the task being tested
- `tested_branch` — the branch tested
- `tested_commit` — the commit hash tested
- `status` — `passed` | `failed` | `blocked`
- `commands_run` — list of commands executed during testing
- `evidence` — output logs, screenshots, or other evidence
- `bugs` — list of bug IDs if any failures were found
- `regression_risks` — any regression risks identified

## Boundary Rules

- Testers must not edit business code.
- Executors and testers must not write files under `.ai-control/`; they return structured output to the orchestrator.
- The orchestrator is responsible for persisting task boards, bug boards, handoffs, reports, and state updates.
- A task may be marked complete only after verification evidence is present.
