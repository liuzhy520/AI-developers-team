# Project Instructions

## Purpose

This repository defines a multi-agent orchestration workflow for GitHub Copilot in VS Code.

## Core Commands

- No build step is required for the repository structure itself.
- Review markdown and instruction consistency after workflow changes.
- Prefer updating canonical instructions and templates together.

## Architecture

- `.github/` contains global instructions, scoped instructions, and agent definitions.
- `.copilot/skills/multi-agent-orchestrator/` contains the skill contract and prompt templates.
- `.ai-control/` contains runtime workflow state and durable context memory.
- `docs/` and `README.md` explain the workflow to humans.

## Conventions

- Treat `.ai-control/session.json` as the source of truth.
- Prefer JSON for machine-updated workflow artifacts.
- Keep handoffs human-readable in markdown.
- Keep instructions concise and operational, not aspirational.