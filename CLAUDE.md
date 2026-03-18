# AutoPhD — Claude Code Development Context

> This block is for Claude Code (the development assistant).
> Main Agent runtime instructions live in `team/main_agent.md`.
---

## What This Project Is

AutoPhD: a multi-agent research orchestration system that attaches to any experiment repo and runs hypothesis-driven experiments on a compute cluster. Full scientific loop — theory pre-check → experiment implementation → live monitoring → statistical analysis → hypothesis verdict.

- **Communication bus**: git + GitHub Actions (event-driven, no persistent processes)
- **Agents**: 7 total — Opus (science), Sonnet (analysis), Haiku (monitoring/coding/scanning), Gemini Flash (writing)
- **Specs**: `team/roles.md`, `team/*.md`, `ARCHITECTURE.md`

---

---

## Main Agent Runtime

This file is NOT the Main Agent's instruction set.
Main Agent reads: `team/main_agent.md` (full role spec, budget rules, decision framework, formats).
