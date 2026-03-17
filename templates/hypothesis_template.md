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

{List every metric to be logged. Each line becomes a column in diagnostics.csv.}

### Primary (must compute)
- `regret`: normalised simple regret at each iteration, per seed
- `best_y`: best function value seen so far, per seed

### Method-specific diagnostics
- `spread_metric`: Σᵢ∈S aᵢ²Lᵢ² per RAASP draw
- `subspace_angle`: angle between A(x_best) and A(x_0) per starting point (degrees)
- `lambda_1`: largest eigenvalue of G_x at x_best
- `lambda_2`: second eigenvalue of G_x at x_best
- `rho`: λ₁/(λ₁+λ₂), effective Fisher rank

### Standard diagnostics
- `z_score`: (μ(x_best) - f*) / σ(x_best)
- `trust_region_size`: current TuRBO trust region side length
- `n_successes`, `n_failures`: TuRBO counters
- `lengthscale_mean`, `lengthscale_min`, `lengthscale_max`: GP ARD lengthscales

## Stopping Rules

{Rules for health_check.py. Format: YAML. Each rule becomes a custom check.}

```yaml
stopping_rules:

  early_stop_method_losing:
    description: "Stop if method is 20% worse than baseline past midpoint"
    trigger: "iteration > budget/2 AND mean_regret_method > mean_regret_baseline * 1.2"
    severity: red
    window: 20  # rows to average over

  early_stop_nan:
    description: "Any NaN in regret"
    trigger: "any(isnan(regret))"
    severity: red
    window: 1

  warn_rank1_collapse:
    description: "ρ approaching 1 — exploitation/exploration conflated"
    trigger: "mean(rho_last_10) > 0.95"
    severity: yellow
    window: 10

  warn_spread_degrading:
    description: "Spread metric declining over last 30 iterations"
    trigger: "spread_metric_slope_last_30 < -0.01"
    severity: yellow
    window: 30

  warn_plateau:
    description: "Regret not improving"
    trigger: "regret_slope_last_30 > -0.001"
    severity: yellow
    window: 30
```

## Statistical Analysis Plan

```yaml
primary_test:
  metric: normalised_regret
  aggregation: mean_over_seeds
  evaluation_point: t=200
  test: wilcoxon_signed_rank
  paired: true  # pair by seed
  alpha: 0.1
  effect_size_threshold: 0.15  # 15% improvement threshold

secondary_analyses:
  - regret_curves: mean ± SE at every iteration
  - spread_ratio: mean(SA-RAASP spread) / mean(uniform spread)
  - spectral_trajectory: λ₁, λ₂, ρ vs iteration
  - correlation: spearman(rho_at_t20, final_regret)

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
