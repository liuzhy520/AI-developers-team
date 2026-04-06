---
applyTo: ".ai-control/**"
---

# Executor and Tester Contract

These rules apply when working with `.ai-control/` workflow state.

## Executor Output

After completing a task, the Executor updates its own task in `session.json`:

- `status` — `"done"` or `"blocked"`
- `changed_paths` — files modified
- `verified` — array of verification results: `{ by, command, exit_code, summary, at }`
- `notes` — observations, risks, or blockers

The Executor consumes but does not modify these routing fields:

- `complexity`
- `selected_models`
- `execution_profile`
- `dispatch_strategy`

The Executor does NOT update: `log`, `decisions`, `phase`, `mode`, other tasks, or task `issues`.

## Tester Report

The Tester returns a structured JSON report to the Orchestrator. The Tester does NOT modify any files.

Report must include:

- `task_id` — the task tested
- `status` — `"passed"`, `"failed"`, or `"issues_found"`
- `verification_rerun` — results of re-running Executor's verification commands
- `independently_verified` — what the Tester checked beyond the Executor's tests
- `edge_cases_tested` — adversarial scenarios tested with results
- `code_review_findings` — issues found by reading the code
- `issues_found` — bugs or gaps discovered, with evidence
- `suggested_test_code` — test code the Executor should implement
- `regression_risks` — identified regression risks

Tester prompts should also include the task's `complexity`, `execution_profile`, and selected tester model so the Tester can calibrate depth without widening scope.

## Tester Requirements

- Tester MUST independently verify — not just re-run the Executor's commands.
- Tester MUST design adversarial test scenarios based on the actual code changes.
- Tester MUST NOT report "all passed" without documenting independent testing evidence.
- Tester MUST NOT modify any file — business code, test code, config, or `.ai-control/`.
- Tester SHOULD assume a higher-capability review mode for medium/high complexity tasks and multi-round retests.

## Boundary Rules

- The Orchestrator is the only agent that persists Tester results into `session.json`.
- The Orchestrator arbitrates issues: decides severity, assigns fixes to Executor.
- A task may be marked `"done"` only after the Tester has reviewed it.
- If the Tester triggers a second fix→retest loop or finds a high-severity issue, the Orchestrator should consider escalating the Executor model tier and record that decision in the task history.
