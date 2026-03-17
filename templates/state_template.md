# State File Template

Copy this to `experiments/{run-id}/state.md` at experiment creation.  
This file is append-only. Never delete or edit past timeline entries.

---

# Experiment {run-id} State

## Metadata

| Field | Value |
|-------|-------|
| **Hypothesis** | H{NNN}: {one-line claim} |
| **Status** | PLANNED |
| **Created** | {timestamp} |
| **Last updated** | {timestamp} |
| **Method** | {method_name} |
| **Baseline** | {baseline_name} |
| **Benchmark** | {benchmark_name}, d={dim} |
| **Budget** | {budget} iters × {n_seeds} seeds |
| **Timeout** | {n} min |

Valid status values: `PLANNED → SETUP → RUNNING → PAUSED → ANALYSING → COMPLETE → FAILED → KILLED`

---

## Hypothesis (self-contained copy)

**Claim:** {one sentence}

**Prediction:** {quantitative prediction from hypothesis file}

**Falsification:** {what result would disprove this}

**Primary metric:** {metric name, e.g., "normalised regret at t=200"}

---

## Configuration

{Paste the relevant fields from config.json here as a readable summary.}

---

## Timeline

*Entries are added chronologically. Do not edit past entries. Add new entries at the bottom.*

---

### {timestamp} — Experiment created [Main Agent]

{What was set up and why. Reference the hypothesis file. Note any decisions made during setup.}

---

### {timestamp} — Theory pre-check [Main Agent]

**Theorems cited in hypothesis:** {list}

**Logical chain:** {claim} → {mechanism} → {prediction}

**Gaps identified:** {any gap between what the theorem proves and what the experiment claims, or "None"}

**Pre-check verdict:** {SOUND | FLAWED | INCOMPLETE}

{If FLAWED or INCOMPLETE: specific issue and resolution required before launch.}

---

### {timestamp} — Experiment Agent instructed [Main Agent]

**Task:** {one-sentence description of what Experiment Agent was asked to do}

**Expected outputs:** {file list}

---

### {timestamp} — Code review [Experiment Agent]

**Status:** {PASSED | FAILED | QUESTIONS_PENDING}

**Checks passed:**
- [ ] log.csv schema matches template
- [ ] diagnostics.csv schema matches hypothesis
- [ ] interrupt.json polling implemented
- [ ] custom_checks.py implements all stopping rules
- [ ] Smoke test passed

**Issues found:** {list or "None"}

**Questions for Main Agent:** {list or "None"}

---

### {timestamp} — Launched on cluster [Human / Watcher]

**Launch command:** `python run_experiment.py {path/to/config.json}`

**Cluster:** {machine name or "local"}

**Expected completion:** {timestamp}

---

### {timestamp} — Monitor check-in #{n} (iteration ~{n}) [Monitor Agent]

**Trigger:** {YELLOW_HEARTBEAT | HUMAN_CHECKIN}

**Health:** {GREEN | YELLOW | RED}

**Regret trajectory:** {DECREASING | PLATEAU | DIVERGING}

**Key numbers:**
- Method regret (mean ± SE): {value}
- Baseline regret (mean ± SE): {value}
- ρ (mean last 5): {value}
- Spread metric trend: {IMPROVING | STABLE | DEGRADING}

**Flags fired:** {list or "None"}

**Decision:** {CONTINUE | ESCALATE | INVESTIGATE}

*Full report: `monitor_report.md`*

---

### {timestamp} — [OPTIONAL: Investigation] [Main Agent]

{Only add this entry if a problem was found and investigated.}

**Problem:** {what Monitor Agent detected}

**Diagnosis:** {what it means, theoretically}

**Action:** {what was changed — config field, code fix, or kill decision}

**Reasoning:** {why this action, not another}

---

### {timestamp} — [OPTIONAL: Interrupt + relaunch] [Main Agent]

**Interrupt set:** `interrupt.json: {"interrupt": true, "reason": "{reason}", "relaunch": true}`

**Config changes:**
- {field}: {old value} → {new value}
- {field}: {old value} → {new value}

**Expected effect:** {what should change in the next run}

---

### {timestamp} — Experiment completed [Watcher]

**Status:** {COMPLETE | INTERRUPTED}

**Iterations completed:** {n}/{budget} ({pct}%)

**Seeds completed:** {n}/{total}

**Elapsed time:** {n} minutes

**Artifact bundle:** `artifact_bundle.md`

---

### {timestamp} — Analyst report received [Analyst Agent]

**Primary result:**
- Method regret (t=200): {value} ± {SE}
- Baseline regret (t=200): {value} ± {SE}
- Effect size: {pct}% ({direction})
- p-value (Wilcoxon): {value}

**Statistical verdict:** {THRESHOLD_MET | THRESHOLD_NOT_MET | INCONCLUSIVE}

*Full report: `final_report.md`*

---

### {timestamp} — Theory post-check [Main Agent]

**Key result:** {the one number that matters}

**Theory prediction:** {what the relevant theorem predicted}

**Consistency:** {CONSISTENT | INCONSISTENT | UNEXPECTED_BUT_EXPLAINABLE}

**Mathematical assessment:** {3–5 sentences}

**Open question for follow-up:** {one specific question this result raises}

---

### {timestamp} — History condensed [Summarizer Agent]

**Summary written:** `history/{run-id}_summary.md`

**Mem0 entry stored:** {YES | NO (reason)}

---

## Hypothesis Verdict

*Written by Main Agent after all checks are complete.*

**Status:** {SUPPORTED | REJECTED | INCONCLUSIVE | PARTIALLY SUPPORTED}

**Key number:** {primary metric — include value, SE, p-value, and n}

**Predicted threshold:** {from hypothesis file}

**Theory consistency:** {CONSISTENT | INCONSISTENT | UNEXPECTED_BUT_EXPLAINABLE}

**Summary:** {One paragraph. What happened. What it means. What to do next.}

---

## Current Assessment

*Updated by Main Agent on every write. Reflects the present moment, not history.*

{1–2 sentences: where things stand RIGHT NOW.}

---

## Open Questions

*Updated throughout the experiment. Remove questions when resolved.*

1. {Question — added by whom, at what timestamp}
2. {Question}

---

## Corrections

*Any overclaims by subagents that were corrected.*

{Leave blank if none.}
