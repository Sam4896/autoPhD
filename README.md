# autoresearch

> An agentic research orchestration system for running, monitoring, and analysing ML experiments.  
> Inspired by [karpathy/autoresearch](https://github.com/karpathy/autoresearch) and the agent harness pattern.

---

## What this is

`autoresearch` is a structured multi-agent system where a **Main Agent** (Opus) orchestrates a team of **Subagents** (Sonnet/Cursor) to run scientific experiments end-to-end — from hypothesis to conclusion — without requiring continuous human supervision.

The system is designed around one core constraint: **the expensive model never watches logs**. It only reads structured summaries produced by cheap models or lightweight scripts. Token spend is proportional to events (problems, completions, surprises), not to wall-clock experiment time.

---

## Design Philosophy

| Principle | How it's enforced |
|-----------|------------------|
| Token efficiency | Main Agent reads summaries only; raw logs go to Sonnet |
| One hypothesis at a time | Each experiment maps to exactly one `H{NNN}.md` |
| Auditable | Every agent decision is timestamped in `state.md` |
| Interruptible | `interrupt.json` allows clean stop/relaunch at any point |
| Memory across experiments | Mem0 stores reasoning, not just results |
| Dumb cluster, smart agents | `health_check.py` has no LLM; logic lives in hypothesis rules |

---

## Repository Structure

```
autoresearch/
├── README.md                          # This file
├── CLAUDE.md                          # Main Agent (Opus) master instructions
├── .cursorrules                       # Subagent (Sonnet/Cursor) instructions
│
├── team/                              # Agent role definitions (read by agents at boot)
│   ├── roles.md                       # Quick map of all roles
│   ├── main_agent.md                  # Orchestrator / Mathematician (Opus)
│   ├── experiment_agent.md            # Coder / Implementer (Sonnet)
│   ├── monitor_agent.md               # Log Reader (Sonnet)
│   ├── analyst_agent.md               # Result Analyst (Sonnet)
│   └── summarizer_agent.md            # State & Memory Writer (Sonnet)
│
├── templates/                         # Canonical file schemas
│   ├── hypothesis_template.md         # How to write a hypothesis file
│   ├── state_template.md              # state.md structure
│   ├── log_schema.md                  # log.csv and diagnostics.csv schema
│   └── heartbeat_schema.md            # heartbeat.json schema + stopping rule DSL
│
├── memory/                            # Mem0 integration
│   ├── memory_schema.md               # What to store and how
│   └── mem0_config_template.json      # Mem0 API config template
│
├── cluster/                           # Cluster-side scripts (no LLM)
│   └── README.md                      # health_check.py design + heartbeat spec
│
└── projects/
    └── riemannian-bo/                 # First research project
        ├── project.md                 # Domain context, benchmarks, eval metrics
        ├── context/
        │   ├── theorems.md            # Key theorems the Main Agent can reference
        │   └── prior_work.md          # Literature summary
        ├── hypotheses/
        │   └── H001_SA_RAASP.md       # First hypothesis: SA-RAASP vs uniform RAASP
        ├── experiments/               # Created at runtime per run
        ├── history/                   # Condensed summaries of completed experiments
        └── src/                       # Project-specific experiment code
```

---

## Agent Team

| Agent | Model | Role | When invoked |
|-------|-------|------|--------------|
| **Main Agent** | Opus | Orchestrator + Mathematician | On events only (setup, interrupt, completion) |
| **Experiment Agent** | Sonnet | Coder + Implementer | When Main Agent issues instructions |
| **Monitor Agent** | Sonnet | Log Reader | On heartbeat yellow/red or human check-in |
| **Analyst Agent** | Sonnet | Result Analyst | On experiment completion |
| **Summarizer Agent** | Sonnet | State + Memory Writer | After every significant event |

---

## Flow Overview

```
You write H001.md
    → Main Agent reads it → creates config.json + state.md
        → Experiment Agent writes experiment code
            → Cluster runs; health_check.py writes heartbeat.json
                → [green] nothing happens
                → [yellow] Monitor Agent reads logs → monitor_report.md
                → [red] interrupt.json set → experiment stops
                    → Summarizer Agent writes artifact report
                        → Main Agent reads report → reasons → stores in Mem0
                            → issues new instructions → Experiment Agent relaunches
        → [completion] Analyst Agent writes final_report.md
            → Summarizer Agent condenses state.md → history/
                → Main Agent writes verdict → stores in Mem0
```

---

## Getting Started

1. Read `CLAUDE.md` if you are the Main Agent.
2. Read `.cursorrules` if you are a subagent.
3. Read `team/roles.md` to understand your collaborators.
4. Read the project's `project.md` for domain context.
5. The active hypothesis is in `projects/{project}/hypotheses/`.

---

## Current Project: Riemannian BO

Testing whether **Spectral-Adaptive RAASP (SA-RAASP)** — perturbing TuRBO starting points along the eigenvectors of the pullback Fisher metric $G_x$ rather than random coordinate axes — achieves lower regret on non-axis-aligned high-dimensional benchmarks.

Active hypothesis: [`H001_SA_RAASP.md`](projects/riemannian-bo/hypotheses/H001_SA_RAASP.md)
