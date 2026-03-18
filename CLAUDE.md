# AutoPhD — Claude Code Development Context

> This block is for Claude Code (the development assistant).
> Main Agent runtime instructions live in `team/main_agent.md`.

---

---

## What This Project Is

AutoPhD: a multi-agent research orchestration system that attaches to any experiment repo and runs hypothesis-driven experiments on a compute cluster. Full scientific loop — theory pre-check → experiment implementation → live monitoring → statistical analysis → hypothesis verdict.

- **Communication bus**: git + GitHub Actions (event-driven, no persistent processes)
- **Agents**: 7 total — Opus (science), Sonnet (analysis), Haiku (monitoring/coding/scanning), Gemini Flash (writing)
- **Specs**: `team/roles.md`, `team/*.md`, `ARCHITECTURE.md`

---

## Project State TLDR

*Append one crisp line after each development session.*

- 2026-03-18: Initial architecture — 7-agent team, Haiku for all code writing, git/GitHub Actions bus, no Cursor.
- 2026-03-18: Removed Cursor; Experiment + Bug Fixer Agents now claude-haiku-4-5 via API; Analyst can WebFetch user-provided refs.
- 2026-03-18: `.memories/` two-layer memory system — memory_tools.py (stdlib, 14 commands), episodic + semantic layers, CLAUDE.md streamlined.
- 2026-03-18: Fixed session ID detection bug (use JSONL filename stem); added `sync-from-history` command to auto-import all local CLI sessions into index.
- 2026-03-18: Removed `.memories/` chat-tracking system; rely on Claude Code built-in auto-memory for all chat/session context.

---

### After session ends

Append one line to **Project State TLDR** above.

---

## Main Agent Runtime

This file is NOT the Main Agent's instruction set.
Main Agent reads: `team/main_agent.md` (full role spec, budget rules, decision framework, formats).
