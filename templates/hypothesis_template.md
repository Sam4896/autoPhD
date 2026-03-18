# Hypothesis Template

Copy this file to `projects/{project}/hypotheses/H{NNN}_{short_name}.md`.  
Fill every section. Do not leave placeholders. Incomplete hypothesis files will not be launched.

---

# H{NNN}: {Short descriptive title}

## Claim

{One sentence. The simplest possible statement of what you believe to be true.  
Bad: "SA-RAASP is better."  
Good: "Perturbing TuRBO starting points along the top-2 eigenvectors of G_x achieves lower normalised regret than uniform RAASP on benchmarks where the active subspace is not axis-aligned."}

## Mechanism

{2–4 sentences explaining WHY the claim should be true. This is the theoretical story.  
It should be specific enough that a failed experiment could identify exactly which part of the mechanism is wrong.}

## Prediction

{One quantitative statement. Must include:
- The metric (e.g., normalised regret at t=200)
- The threshold (e.g., ≥15% improvement)
- The comparison condition (e.g., vs TuRBO with uniform RAASP)
- The benchmark and conditions (e.g., Hartmann-6/100D rotated)
- The aggregation (e.g., mean over 20 seeds)}

## Falsification

{What result would disprove this hypothesis? Must be quantitative.  
Bad: "If SA-RAASP doesn't improve things."  
Good: "If the effect size is < 5% or p > 0.1 (Wilcoxon signed-rank, n=20 seeds) on the rotated benchmark."}

## Experiment Plan

```yaml
treatment:
  method: {name of the method being tested}
  description: {one sentence — what makes it different}

control:
  method: {name of baseline}
  description: {one sentence — what standard method is being compared to}

benchmark:
  name: {benchmark function name}
  dimension: {d}
  variant: {axis-aligned | rotated | embedded | full}
  rotation: {random Q ∈ O(d) with seed {seed} | none}

budget:
  evaluations_per_seed: {n}
  seeds: {n}
  initial_sobol: {n}
  sequential: {n}

kernel: {SE-ARD | Matern52-ARD | ...}
acquisition: {LogEI | EI | UCB | ...}
timeout_minutes: {n}
interrupt_poll_every: {n_iterations}
```

## Diagnostics to Track

{List every metric to be logged. Each line becomes a column in diagnostics.csv.
The Monitor Agent and Analyst Agent read column names directly from the data — be precise here.}

### Primary (must compute — always include these two)
- `{primary_metric}`: {description — e.g., "normalised simple regret at each iteration, per seed"}
- `best_y`: best objective value seen so far, per seed

### Method-specific diagnostics
{Add any internal quantities of your method that help diagnose its behaviour.}
- `{diagnostic_column_1}`: {description and units}
- `{diagnostic_column_2}`: {description and units}
- (add as many as needed — each becomes a column in diagnostics.csv)

### Standard diagnostics
{Quantities that are informative regardless of the method.}
- `{standard_column_1}`: {description}
- `{standard_column_2}`: {description}

## Stopping Rules

{Rules for health_check.py. Format: YAML. Each rule becomes a custom check.
Use column names exactly as defined in ## Diagnostics to Track above.}

```yaml
stopping_rules:

  early_stop_method_losing:
    description: "Stop if method is {X}% worse than baseline past midpoint"
    trigger: "iteration > budget/2 AND mean_{primary_metric}_method > mean_{primary_metric}_baseline * {threshold}"
    severity: red
    window: 20  # rows to average over

  early_stop_nan:
    description: "Any NaN in primary metric"
    trigger: "any(isnan({primary_metric}))"
    severity: red
    window: 1

  warn_diagnostic_threshold:
    description: "{Description of what this diagnostic measures and why it matters}"
    trigger: "{condition on a diagnostic column from ## Diagnostics to Track}"
    severity: yellow
    window: 10

  warn_plateau:
    description: "Primary metric not improving"
    trigger: "{primary_metric}_slope_last_30 {> or < threshold depending on min/max}"
    severity: yellow
    window: 30

  # Add more rules as needed. Each rule fires health_checker.py → heartbeat commit → brain_listen.yml
```

## Statistical Analysis Plan

{The Analyst Agent executes exactly what is written here. Be precise. Specify metric names as they appear in log.csv.}

```yaml
primary_test:
  metric: {column name from log.csv — e.g., normalised_regret}
  aggregation: mean_over_seeds
  evaluation_point: {e.g., t=200, or "final_iteration"}
  test: {e.g., wilcoxon_signed_rank | paired_t_test | permutation_test}
  paired: true  # pair by seed
  alpha: {significance threshold, e.g., 0.1}
  effect_size_threshold: {minimum meaningful improvement, e.g., 0.15 = 15%}

secondary_analyses:
  - {name}: {description — e.g., "regret curves: mean ± SE at every iteration"}
  - {name}: {description — e.g., "correlation: spearman({diagnostic_col}_at_t{n}, final_{metric})"}
  # Add as many secondary analyses as needed.
  # The Analyst Agent will not compute anything not listed here.

bootstrap:
  n_samples: 10000
  resample_unit: seed
  confidence_level: 0.95
```

## Relevant Theory

{List theorems from context/theorems.md that this hypothesis depends on.  
For each, state the connection.}

- **{Theorem name / number}**: {How it supports or motivates this hypothesis}
- **{Theorem name / number}**: {How it supports or motivates this hypothesis}

Also note any gap between the theorem and the experiment:
- **Gap**: {If the theorem is first-order only, if it assumes something the experiment doesn't satisfy, etc.}

## Dependencies

{What must be true for this experiment to be interpretable?  
If this is your first experiment, write "None."  
If this experiment depends on results from H{NNN}, state it here.}

## Expected Difficulty

{LOW | MEDIUM | HIGH}
Reasoning: {one sentence}

## Open Questions

{Anything you are uncertain about that might affect the design.
These are questions for the Main Agent to resolve before launch.}

1. {Question 1}
2. {Question 2}

## References

{Optional. Provide any papers or GitHub repositories the agent should read for insights.
In GUIDED mode, the Main Agent will fetch these URLs directly (WebFetch, not WebSearch).
In EXPLORE mode, these are also used as seeds for further literature search.

The Security Agent scans all fetched content before it reaches the Main Agent.
Be specific about what to extract — agents will not read the full paper, only relevant sections.}

### Papers

- {URL or DOI} — {what to extract: e.g., "Theorem 3.2 — the convergence bound under compactness assumption"}
- {URL or DOI} — {what to extract: e.g., "Section 4 — hyperparameter sensitivity analysis for this benchmark"}

### GitHub Repositories

- {repo URL} — {what to look for: e.g., "implementation of the baseline method in src/baselines/; compare to our method"}
- {repo URL} — {what to look for: e.g., "benchmark definition and rotation matrix seed"}

### Notes for the Agent

{Any specific instructions for interpreting the references.
Example: "The paper uses a different notation: their 'f' is our 'regret'. Their Theorem 2 is the one we extend."
If no references: delete this section.}

---

## AutoPhD Integration Settings

These fields are read by `brain/connect.py` and `brain/invoke.py`.

```yaml
autoPhD:
  # GUIDED: Brain works only with files you provided. No internet access.
  # EXPLORE: Brain may call WebSearch with justification gate (you must commit this change yourself).
  exploration_mode: GUIDED

  # Which files/directories the Experiment Agent may modify
  allowed_paths:
    - src/methods/
    - src/config/

  # Files/directories the Experiment Agent must never touch
  protected_paths:
    - src/benchmarks/
    - data/
    - run_experiment.py
    - .github/

  # Human approval required before any code change is committed
  # Set to "autonomous" to allow agents to commit directly (agents will still notify you)
  approval_mode: approval_required

  # Maximum experiment relaunches (budget guard)
  n_experiment_trials: 5

  # Cost limit in USD across all agent calls for this hypothesis
  max_cost_usd: 10.00
```
