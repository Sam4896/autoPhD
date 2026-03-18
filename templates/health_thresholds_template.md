# Health Thresholds — {project_name} / {hypothesis_id}

Copy this file to `projects/{project}/health_thresholds.md` and fill it in before running `autoPhD connect`.

This file is read by `brain/connect.py` to generate `health_checker.py` for the experiment repo. Every section must be filled in. Incomplete files will not generate a valid health checker.

---

## How this file is used

1. `autoPhD connect` reads this file
2. It generates `health_checker.py` from `templates/health_checker.py.j2` using your thresholds
3. `autoPhD_adapter.py` runs `health_checker.py` in a background thread
4. The health checker reads your experiment logs and applies your thresholds
5. On a health event, it commits `heartbeat.json` to the experiment branch
6. The brain listens for that commit via GitHub Actions

**Write specific, quantitative thresholds. Vague descriptions (e.g., "if it looks bad") cannot be translated into code.**

---

## Primary Metric

The main metric that your experiment optimises. The health checker tracks this first.

```yaml
primary_metric:
  name: regret                        # column name in log.csv
  direction: minimize                  # minimize | maximize
  # The value that constitutes good performance (lower is better for minimize)
  # Leave null if no absolute target — relative checks below will still apply
  absolute_target: null                # e.g., 0.1 — or null
```

---

## Log File Locations

Where your experiment writes its output files. These are relative to the experiment working directory.

```yaml
log_files:
  primary_log: log.csv                 # required — the per-iteration metrics log
  diagnostics_log: diagnostics.csv     # optional — method-specific diagnostics
  # Column in primary_log that tracks iteration number
  iteration_column: iteration
  # Column in primary_log that tracks which seed is running
  seed_column: seed
```

---

## Green Health Criteria

The experiment is healthy when ALL of these are true.
If any green criterion is violated, status becomes YELLOW.

```yaml
green_criteria:
  # Primary metric is improving (or within acceptable range)
  - name: metric_improving
    description: "{primary_metric} is decreasing (improving) over recent iterations"
    condition: "slope of {primary_metric} over last {window} iterations < {threshold}"
    window: 30          # number of iterations to look back
    threshold: -0.001   # slope must be more negative than this (for minimize direction)
    # Set to null to disable this check
    enabled: true

  # No missing or invalid values
  - name: no_nan
    description: "No NaN or Inf in any numeric column"
    condition: "no NaN or Inf in primary_log"
    enabled: true

  # All seeds are running
  - name: all_seeds_active
    description: "All expected seeds have data in the current iteration"
    condition: "number of unique seeds in last {window} rows >= {min_seeds}"
    window: 10
    min_seeds: 20       # set to your n_seeds value
    enabled: true

  # Memory / resource (optional — leave enabled: false if not applicable)
  - name: resource_ok
    description: "No resource exhaustion warnings in log"
    condition: "status column does not contain 'oom' or 'memory'"
    enabled: false
```

---

## Yellow Health Criteria (Warning — do not stop automatically)

These conditions indicate the experiment may be struggling but should continue.
When any yellow criterion fires, a YELLOW heartbeat is committed.
The brain will read the heartbeat and may invoke the Monitor Agent.

```yaml
yellow_criteria:
  # Primary metric has plateaued
  - name: plateau_warning
    description: "Primary metric not improving — may indicate convergence or stagnation"
    condition: "slope of {primary_metric} over last {window} iterations is between {lower} and {upper}"
    window: 30
    lower: -0.001       # not improving faster than this
    upper: 0.001        # and not getting much worse either (flat)
    enabled: true

  # High variance across seeds
  - name: high_variance
    description: "High variance in primary metric across seeds — possible instability"
    condition: "std({primary_metric}) / mean({primary_metric}) > {threshold}"
    threshold: 0.5      # coefficient of variation > 50%
    window: 10          # rows to average over
    enabled: true

  # Experiment approaching timeout
  - name: timeout_risk
    description: "Experiment may not complete before timeout"
    condition: "elapsed_time / timeout_minutes > {threshold}"
    threshold: 0.8      # 80% of time used
    enabled: true

  # Method-specific warning (fill in or disable)
  - name: method_specific_warning_1
    description: "{Describe what diagnostic value you want to watch}"
    # Example: "rank_collapse_warning: rho (lambda1/(lambda1+lambda2)) > 0.9"
    condition: "{column_name} {operator} {threshold}"
    column_name: rho           # column in diagnostics_log
    operator: ">"
    threshold: 0.9
    window: 10
    enabled: false             # set to true and fill in if applicable

  # Add more method-specific yellow criteria here
```

---

## Red Health Criteria (Critical — stop the experiment immediately)

When any red criterion fires, the experiment is interrupted gracefully and a RED heartbeat committed.
The brain will immediately invoke the Monitor Agent and then the Main Agent.

```yaml
red_criteria:
  # NaN in primary metric — always red
  - name: nan_in_metric
    description: "NaN or Inf in primary metric — experiment is broken"
    condition: "any NaN or Inf in {primary_metric} column"
    enabled: true

  # Method is clearly losing — early stop to save resources
  - name: method_clearly_losing
    description: "Method regret significantly worse than baseline past midpoint"
    condition: >
      iteration > budget * 0.5
      AND mean({method_column}) / mean({baseline_column}) > {ratio_threshold}
      (averaged over last {window} iterations)
    method_column: null        # set to your method name column or leave null to disable
    baseline_column: null      # set to your baseline name column
    ratio_threshold: 1.2       # method is 20% worse than baseline
    window: 20
    enabled: false             # set to true if you have method vs baseline comparison

  # Experiment crashed (exception in log)
  - name: crash_detected
    description: "Status column shows error or exception"
    condition: "status column contains 'error' or 'exception' or 'crash'"
    enabled: true

  # Method-specific red criterion (fill in or disable)
  - name: method_specific_red_1
    description: "{Describe critical failure condition}"
    condition: "{column_name} {operator} {threshold}"
    column_name: null
    operator: ">"
    threshold: null
    window: 1
    enabled: false

  # Add more method-specific red criteria here
```

---

## Experiment Success Criteria (early stop if met — saves resources)

When any success criterion is met, the experiment exits cleanly with `status: "SUCCESS_EARLY_STOP"`.
The Analyst Agent is still invoked to produce the final report.

```yaml
success_criteria:
  # Primary metric has reached target (if applicable)
  - name: absolute_target_reached
    description: "Primary metric has reached the defined target value"
    condition: "mean({primary_metric}) over last {window} rows <= {target}"
    target: null           # set to your success threshold, or null to disable
    window: 5
    enabled: false         # only enable if you have an absolute target

  # Hypothesis metric threshold exceeded
  - name: hypothesis_threshold_met
    description: "Effect size vs baseline has exceeded hypothesis threshold"
    # Example: "method achieves ≥15% improvement over baseline for 5+ consecutive checkpoints"
    condition: >
      (mean({baseline_column}) - mean({method_column})) / mean({baseline_column})
      >= {effect_threshold}
      for last {consecutive_checkpoints} checkpoints
    effect_threshold: 0.15    # 15% improvement threshold from hypothesis
    consecutive_checkpoints: 5
    method_column: null        # fill in your method column name
    baseline_column: null      # fill in your baseline column name
    enabled: false             # enable if you have method vs baseline comparison

  # Add more success criteria here
```

---

## Commit Message Format

The health checker commits heartbeats with structured messages. You can customise the format:

```yaml
commit_messages:
  green: "health: GREEN iter={iteration}/{budget} {primary_metric}={value:.4f}"
  yellow: "health: YELLOW iter={iteration}/{budget} flag={flag_name}"
  red: "health: RED iter={iteration}/{budget} flag={flag_name}"
  success: "complete: iter={iteration}/{budget} status=SUCCESS_EARLY_STOP"
  interrupted: "complete: iter={iteration}/{budget} status=interrupted"
  normal_complete: "complete: iter={iteration}/{budget} status=success"
```

---

## Check Interval

How frequently the health checker reads the log and evaluates thresholds:

```yaml
health_check_interval_seconds: 30   # how often to run health checks
git_pull_interval_seconds: 30       # how often to pull for interrupt.json
```

Lower values = faster response to problems, but more git operations.
30 seconds is the recommended default for cluster experiments.
