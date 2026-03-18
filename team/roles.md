# Team Roles — AutoPhD v1.0

Quick reference for all 7 agents + Cursor Code Agent. Each has a detailed spec in its own file.
See `ARCHITECTURE.md §4` for the full agent hierarchy diagram.

---

## Role Map

```
Human (You)
├── Writes hypothesis files and health_thresholds.md
├── Approves launches (runs autoresearch_adapter.py on cluster)
├── Reviews PRs in GitHub / Cursor when approval_mode: approval_required
├── Monitors NOTIFICATION.md and PENDING_APPROVAL.md
└── Makes final call on KILL when agent is uncertain

Main Agent — claude-opus-4-6 [main_agent.md]
├── Reads: summaries, state files, hypothesis, budget.json, Mem0
├── Writes: state.md entries, config.json, interrupt.json, history summaries
├── Decides: continue / interrupt / kill / relaunch / hypothesis verdict
├── Gate: checks budget.json before every subagent instruction
└── Invoked: on events via GitHub Actions brain_listen.yml

Monitor Agent — claude-sonnet-4-6 [monitor_agent.md]
├── Reads: heartbeat.json + last 50 rows log.csv + last 30 rows diagnostics.csv
├── Writes: .autoresearch/monitor_report.md
├── Does: detect patterns, flag anomalies, assess trajectory
├── Limit: max 3000 input tokens, max 1500 output tokens
└── Invoked: on heartbeat YELLOW/RED, or human check-in

Analyst Agent — claude-sonnet-4-6 [analyst_agent.md]
├── Reads: full log.csv, full diagnostics.csv, hypothesis, state.md
├── Writes: .autoresearch/final_report.md
├── Does: compute primary metric, significance tests, bootstrap CI (per hypothesis SAP)
├── Limit: thinking budget 5000 tokens, max 4000 output tokens
└── Invoked: when experiment status → COMPLETE or KILLED

Summarizer Agent — gemini-3.1-flash-lite-preview [summarizer_agent.md]
├── Reads: final_report.md, state.md, monitor reports, verdict
├── Writes: artifact_bundle.md, history/{run-id}_summary.md, agent_dialogue.md entries
├── Does: compress experiment lifecycle, write Mem0 entry, update agent_dialogue.md
├── Model: Gemini Flash — this is pure writing, no reasoning required
└── Invoked: after significant events (interrupt, completion, verdict)

Experiment Agent — claude-haiku-4-5 [experiment_agent.md]
├── Reads: Main Agent instruction block, hypothesis, project.md, config.json, relevant src/ files
├── Writes: src/{files within allowed_paths}, .autoresearch/experiment_completed.json
├── Does: implement experiment code directly via API, run smoke test inline
├── Gate: brain/security.py scans the diff before any commit (inline Python, no LLM cost)
└── Invoked: when Main Agent issues a new implementation instruction

Bug Fixer Agent — claude-haiku-4-5 [bug_analyst_agent.md]
├── Reads: monitor_report.md, heartbeat.json, config.json, src/{identified file}
├── Writes: src/{fixed file within allowed_paths}, .autoresearch/bug_completed.json
├── Does: diagnose bug from monitor report, write minimal fix, run smoke test inline
└── Invoked: on code bug (RED heartbeat from NaN/crash/shape error)

Security Agent — claude-haiku-4-5 [security_agent.md]
├── Reads: web search results (EXPLORE mode), fetched URL content (GUIDED + EXPLORE)
├── Writes: .autoresearch/security_review.md (APPROVED / REJECTED + reason)
├── Does: injection scan on external content before it reaches Main Agent or Analyst Agent
├── Does NOT review code — brain/security.py handles code safety inline (zero LLM cost)
└── Invoked: after any WebSearch call or reference URL fetch
```

---

## Model Routing Summary

| Model | Cost tier | Assigned agents |
|-------|-----------|-----------------|
| claude-opus-4-6 | Expensive | Main Agent only |
| claude-sonnet-4-6 + thinking | Medium | Analyst Agent only |
| claude-haiku-4-5 | Cheap | Monitor, Experiment, Bug Fixer, Security |
| gemini-3.1-flash-lite-preview | Free | Summarizer |
| Python inline (brain/security.py) | Zero | Code safety checks, allowed_paths enforcement |

**Rule**: Scientific judgment + theorem reasoning → Opus. Statistical analysis needing computation → Sonnet + thinking. Operational tasks (monitoring, code writing, scanning) → Haiku. Pure writing + summarization → Gemini. Regex pattern checks → Python inline (no LLM).

---

## Invocation Triggers

| Event | Who runs |
|-------|----------|
| Adapter commits `heartbeat: RED` | brain_listen.yml → Monitor Agent → Bug Fixer Agent → Main Agent |
| Adapter commits `heartbeat: YELLOW` (persistent) | brain_listen.yml → Monitor Agent → Main Agent if escalated |
| Adapter commits `complete:` | brain_listen.yml → Analyst Agent → Summarizer → Main Agent |
| Main Agent issues implementation instruction | brain_listen.yml invokes Experiment Agent → writes code → brain/security.py inline scan → PR or commit |
| Main Agent identifies bug | brain_listen.yml invokes Bug Fixer Agent → writes fix → inline scan → PR or commit |
| Experiment Agent writes experiment_completed.json | brain commits it → brain_listen.yml → Main Agent reads result |
| Bug Fixer Agent writes bug_completed.json | brain commits it → brain_listen.yml → Main Agent reads result |
| Main Agent writes verdict | Summarizer Agent → history condensation |
| Reference URL fetched (GUIDED or EXPLORE) | Security Agent sanitises → Main Agent / Analyst Agent receives |
| Web search requested (EXPLORE mode only) | Security Agent sanitises → Main Agent receives |
| Budget exhausted | brain/invoke.py blocks → NOTIFICATION.md written |
| Kill switch activated | interrupt.json pushed → all actions stop |

---

## What Each Agent NEVER Does

| Agent | Never does |
|-------|-----------|
| Main Agent | Read raw logs, write code, run shell commands, self-activate EXPLORE mode |
| Monitor Agent | Interpret results scientifically, write code, update config |
| Analyst Agent | Monitor live experiments, write code, update config |
| Summarizer Agent | Make scientific decisions, write code, run statistical tests |
| Experiment Agent | Make scientific decisions, touch protected_paths, retry smoke test >2 times |
| Bug Fixer Agent | Read full src/ directory, change algorithm logic, update state.md |
| Security Agent | Review code logic, block content based on scientific disagreement, scan code diffs |

---

## Communication Protocol

Agents communicate through **files on the experiment branch**, not through direct calls.

```
Main Agent writes instruction  →  .autoresearch/state.md (INSTRUCTION FOR block)
Subagent reads state.md        →  executes task
Subagent writes output         →  .autoresearch/{report}.md or cursor_task.json
Next invocation reads output   →  Main Agent or another agent reads the report
```

Cursor Code Agent communication uses `cursor_task.json` (brain → Cursor) and `cursor_completed.json` (Cursor → brain).

`brain/invoke.py` is the runtime that:
1. Determines which agent to call (from event type)
2. Checks budget
3. Assembles the input (specific files, capped to token limit)
4. Calls the agent (claude --print or Gemini API)
5. Writes output to the correct file
6. Logs to `agent_comms.jsonl` and `agent_dialogue.md`
7. Commits and pushes updated files to the experiment branch
