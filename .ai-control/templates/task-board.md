# Task Board

## Status Legend

- `todo`: defined but not dispatched
- `in_progress`: currently assigned to an executor
- `blocked`: waiting on dependency or contract resolution
- `ready_for_test`: pushed and waiting for tester
- `failed_test`: tester reported a failure
- `done`: verified and accepted

## Tasks

| Task ID | Title | Status | Owner | Branch | Worktree | Depends On | Tested |
| --- | --- | --- | --- | --- | --- | --- | --- |

## Dispatch Notes

- Only the orchestrator updates this board.
- Keep `Task ID`, `Branch`, and `Worktree` aligned with `state.json` and task cards.
