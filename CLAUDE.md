# CLAUDE.md — P1: LLM Gateway

## What this project is

A multi-provider LLM gateway. Milestone plan: docs/SYLLABUS.md. Full
program context: docs/CHARTERS.md.

## Prime rules

- docs/CHARTERS.md is law. If a request conflicts with the charter (scope, stack, naming, interfaces), refuse and cite the charter line.
- This is a LEARNING project. Optimize for readability over cleverness — clear names, small functions, no magic. I dissect and must understand every line.
- Implement ONLY the current SPEC (docs/specs/current.md). Never build ahead of it, never add unrequested features.
- Always plan first: present your implementation plan and wait for my approval before writing code.
- Small, logical commits with descriptive messages — the commit history is study material.
- After completing a SPEC, write docs/build-notes/<milestone>.md: key decisions, alternatives rejected and why, edge cases, limitations, and 3 questions a reviewer should ask about this code.

## Stack & conventions

- Python: FastAPI, Pydantic v2, uv, ruff, pytest.
- Type hints everywhere. Docstrings explain WHY, not just what.
- Tests for core logic; acceptance criteria in the SPEC map to tests.
- Service listens on :8000, exposes GET /healthz.

## Repo map

- docs/CHARTERS.md — program-wide law (all 9 projects)
- docs/SYLLABUS.md — this project's milestone plan
- docs/STATE.md — current save-game (written by Claude Chat, not you)
- docs/CONCEPTS.md — my mastery ledger (I maintain this)
- docs/CC-SKILLS.md — my Claude Code mastery ledger (I maintain this)
- docs/CC-TRACK.md — Claude Code feature curriculum
- docs/specs/ — one SPEC per milestone; current.md is active
- docs/build-notes/ — your build reports, one per milestone
