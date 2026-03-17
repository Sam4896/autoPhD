# Heartbeat Schema & Stopping Rule DSL

This document specifies the `heartbeat.json` format, the default health checks, and the grammar for custom stopping rules in hypothesis files.

---

## heartbeat.json — full schema

```json
{
  "schema_version": "1.0",
  "timestamp": "ISO 8601",
  "run_id": "string",

  "progress": {
    "iteration": "int — current iteration across all seeds",
    "pct_budget_used": "float — 0–100",
    "elapsed_minutes": "float",
    "timeout_minutes": "float",
    "timeout_risk": "bool — true if elapsed > 0.8 * timeout"
  },

  "seeds": {
    "total": "int",
    "active": "int",
    "crashed": ["list of seed IDs"],
    "crashed_at_iteration": "int or null"
  },

  "regret": {
    "method": {
      "current": "float — mean regret, last 5 rows",
      "stderr": "float",
      "slope_last_10": "float",
      "slope_last_30": "float",
      "trend": "decreasing | plateau | diverging"
    },
    "baseline": {
      "current": "float",
      "stderr": "float",
      "slope_last_10": "float",
      "slope_last_30": "float",
      "trend": "decreasing | plateau | diverging"
    },
    "ratio": "float — method / baseline (null if baseline = 0)"
  },

  "diagnostics": {
    "lambda_1_last": "float or null",
    "lambda_2_last": "float or null",
    "rho_last": "float or null",
    "rho_trend": "stable | increasing | decreasing",
    "spread_metric_last": "float or null",
    "spread_trend": "improving | stable | degrading"
  },

  "data_quality": {
    "nan_rows": "int",
    "nan_columns": ["list of column names with NaN"],
    "missing_rows": "int — expected vs actual",
    "last_timestamp": "ISO 8601"
  },

  "health": "green | yellow | red",

  "flags": [
    "string — each fired default check with trigger value"
  ],

  "custom_flags": [
    "string — each fired custom check with trigger value and message"
  ]
}
```

---

## Default checks (always active, not user-configurable)

These run regardless of what is in the hypothesis file.

| Check | Severity | Condition |
|-------|----------|-----------|
| `nan_in_regret` | RED | Any NaN in the `regret` column |
| `nan_in_diagnostics` | YELLOW | Any NaN in `lambda_1` or `lambda_2` |
| `seed_crash` | RED | `seeds.crashed` is non-empty |
| `all_seeds_crashed` | RED | `seeds.active` == 0 |
| `timeout_imminent` | RED | `elapsed_minutes > 0.9 * timeout_minutes` |
| `data_stale` | RED | Last log write > 5 minutes ago |
| `missing_rows` | YELLOW | `missing_rows > 2 * seeds.total` |

---

## Custom stopping rules DSL

Custom rules are specified in the hypothesis file under `## Stopping Rules`. The Experiment Agent implements them as Python functions in `custom_checks.py`. This section defines what expressions are valid in the `trigger` field.

### Available variables

These variables are available in trigger expressions. They are computed by `health_check.py` from the last N rows of `log.csv` and `diagnostics.csv`, where N is the `window` field.

**From log.csv (window-averaged unless otherwise noted):**

| Variable | Meaning |
|----------|---------|
| `regret` | Mean regret across all seeds |
| `regret_method` | Mean regret for the treatment method |
| `regret_baseline` | Mean regret for the baseline method |
| `regret_slope_last_{N}` | Linear regression slope of regret over last N rows |
| `best_y` | Current best observed value |
| `iteration` | Current iteration number (scalar, not window) |
| `n_seeds_active` | Number of non-crashed seeds |
| `nan_count` | Number of NaN rows in window |

**From diagnostics.csv (window-averaged):**

| Variable | Meaning |
|----------|---------|
| `lambda_1` | Mean λ₁ in window |
| `lambda_2` | Mean λ₂ in window |
| `rho` | Mean ρ in window |
| `spread_metric` | Mean spread metric in window |
| `spread_slope_last_{N}` | Linear regression slope of spread metric |
| `subspace_angle` | Mean subspace angle in window |

**From config:**

| Variable | Meaning |
|----------|---------|
| `budget` | Total budget (iterations × seeds) |
| `n_seeds` | Number of seeds |
| `timeout_minutes` | Timeout in minutes |
| `elapsed_minutes` | Time elapsed (scalar, not window) |

### Allowed operators and functions

```
Arithmetic:  +  -  *  /  **
Comparison:  >  <  >=  <=  ==  !=
Logic:       AND  OR  NOT
Functions:   mean(col)  any(condition)  all(condition)
             isnan(col)  abs(expr)  sqrt(expr)
             slope_last_{N}(col)  — computes linear regression slope
```

### Example valid trigger expressions

```yaml
# Stop if method regret is more than 20% worse than baseline past halfway
"iteration > budget / 2 AND regret_method > regret_baseline * 1.2"

# Stop on any NaN
"any(isnan(regret))"

# Warn on rank-1 collapse
"mean(rho_last_10) > 0.95"

# Warn on degrading spread
"slope_last_30(spread_metric) < -0.01"

# Warn if method is not catching up after 100 iterations
"iteration > 100 AND regret_method > regret_baseline"

# Stop if trust region collapsed
"mean(trust_region_size) < 1e-6"
```

### Invalid expressions (do not use)

```yaml
# No function calls beyond the allowed list
"scipy.stats.ttest_ind(...)"  # INVALID

# No list comprehensions or loops
"[x for x in regret if x > 0]"  # INVALID

# No file I/O
"open('log.csv').read()"  # INVALID

# No variable assignment
"x = regret_method + 1"  # INVALID
```

If your stopping rule is too complex to express in this grammar, implement it as a Python function in `custom_checks.py` instead. The function interface is in `team/experiment_agent.md`.

---

## Health → action mapping

| Health | Auto-action | Human notified? |
|--------|-------------|-----------------|
| GREEN | Nothing | No |
| YELLOW | Watcher flags for optional Monitor check-in | No (unless configured) |
| RED (default check) | Watcher sets `interrupt.json: {"interrupt": true}` | YES — watcher log |
| RED (custom check) | Watcher sets `interrupt.json: {"interrupt": true}` | YES — watcher log |

When the experiment stops due to RED, the Summarizer Agent is invoked automatically by the watcher before the Main Agent is notified. The Main Agent receives the artifact bundle, not the raw interrupt event.

---

## Configuring notification thresholds

In `config.json`, you can override the default auto-interrupt behaviour:

```json
{
  "heartbeat_config": {
    "check_interval_seconds": 30,
    "yellow_auto_monitor": true,
    "red_auto_interrupt": true,
    "red_requires_n_consecutive": 3,
    "notify_on_yellow_after_n_iterations": 20
  }
}
```

- `red_requires_n_consecutive`: how many consecutive RED heartbeats before auto-interrupt (default 1 — interrupt immediately on first RED). Set to 3 to tolerate transient spikes.
- `notify_on_yellow_after_n_iterations`: suppress YELLOW notifications for the first N iterations of an experiment (many experiments are naturally noisy early on).
