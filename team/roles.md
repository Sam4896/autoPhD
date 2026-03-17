# Team Roles

Quick reference. Each agent has a detailed spec in its own file.

---

## Role Map

```
Human (You)
├── Writes hypothesis files
├── Approves launches via SSH / git push
├── Triggers human check-ins when curious
└── Makes final call on KILL vs CONTINUE when agent is uncertain

Main Agent — Opus [main_agent.md]
├── Reads: summaries, state files, hypothesis files, Mem0
├── Writes: state.md entries, config.json, interrupt.json, instructions, history summaries
├── Decides: continue / interrupt / kill / relaunch / hypothesis verdict
└── Invoked: on events (problem detected, experiment complete, human request)

Experiment Agent — Sonnet [experiment_agent.md]
├── Reads: Main Agent instructions, hypothesis file, project.md, existing src/ code
├── Writes: experiment code in src/, config.json fields, custom_checks.py
├── Does: implement, test locally (unit tests only), package for cluster
└── Invoked: when Main Agent issues a coding instruction

Monitor Agent — Sonnet [monitor_agent.md]
├── Reads: log.csv (last N rows), diagnostics.csv (last N rows), heartbeat.json, state.md
├── Writes: monitor_report.md (structured summary — NOT raw data)
├── Does: detect patterns, flag anomalies, assess trajectory
└── Invoked: on heartbeat yellow/red, or human-initiated check-in

Analyst Agent — Sonnet [analyst_agent.md]
├── Reads: full log.csv, full diagnostics.csv, hypothesis file, state.md
├── Writes: final_report.md (statistics, plots spec, interpretation)
├── Does: compute regret curves, run significance tests, compare to predictions
└── Invoked: when experiment status → COMPLETE or KILLED

Summarizer Agent — Sonnet [summarizer_agent.md]
├── Reads: final_report.md, state.md, monitor reports, Main Agent decisions
├── Writes: datetime_experimentname.md (artifact bundle), history/{run-id}_summary.md
├── Does: distil experiment lifecycle into a compact, retrievable record
└── Invoked: after Analyst Agent completes, before Main Agent writes final verdict
```

---

## Invocation Triggers (who causes whom to run)

| Trigger | Who acts |
|---------|----------|
| Heartbeat turns RED | Watcher → sets `interrupt.json` → Summarizer (artifacts) → Main Agent (decision) |
| Heartbeat turns YELLOW | Watcher → Monitor Agent (log read) → Main Agent if escalated |
| Human says "check in" | Monitor Agent → Main Agent if escalated |
| Experiment exits cleanly | Analyst Agent → Summarizer → Main Agent |
| Main Agent issues coding instructions | Experiment Agent |
| Main Agent writes verdict | Summarizer → condenses to history/ |

---

## What each agent NEVER does

| Agent | Never does |
|-------|-----------|
| Main Agent | Read raw logs, write code, run shell commands |
| Experiment Agent | Make scientific decisions, write state.md, touch logs |
| Monitor Agent | Interpret results scientifically, write code, update config |
| Analyst Agent | Monitor live experiments, write code, update config |
| Summarizer Agent | Make scientific decisions, write code, run statistical tests |

---

## Communication protocol

Agents communicate through **files**, not through direct invocation.  
The Main Agent writes an `INSTRUCTION FOR {AGENT}` block (see CLAUDE.md for format).  
The subagent reads the instruction, executes, and writes its output file.  
The Main Agent reads the output file on its next invocation.

No agent should ever try to "call" another agent directly. Files are the message bus.
