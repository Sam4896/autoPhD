# Log Schema

All experiments must write CSV files conforming to these schemas.  
Adding columns is allowed. Removing or renaming required columns is not.

---

## log.csv — Main experiment log

Written every iteration, every seed. One row per (iteration, seed, method).

| Column | Type | Description | Required |
|--------|------|-------------|----------|
| `run_id` | str | Experiment run identifier | YES |
| `seed` | int | Random seed (0-indexed) | YES |
| `method` | str | Method name (matches config) | YES |
| `iteration` | int | BO iteration number (0-indexed) | YES |
| `n_evaluations` | int | Total function evaluations so far | YES |
| `regret` | float | Normalised simple regret: (f* - best_y) / (f* - f_min) | YES |
| `best_y` | float | Best observed function value so far | YES |
| `x_best` | str | JSON of current best point (d-dimensional) | YES |
| `y_new` | float | Function value at the newly evaluated point | YES |
| `x_new` | str | JSON of newly evaluated point | YES |
| `acqf_value` | float | Acquisition function value at x_new | NO |
| `timestamp` | str | ISO 8601 timestamp of this row | YES |
| `status` | str | normal \| interrupted \| complete | YES |

**Notes:**
- `regret` must be in [0, 1]. If the problem has no known optimum, use `best_y` instead and set `regret` to NaN.
- `x_best` and `x_new` are JSON-encoded arrays for portability. Example: `"[0.1, 0.5, 0.9]"`
- `status` is `normal` for all rows except the final row of an interrupted run (`interrupted`) or the final row of a complete run (`complete`).
- Write one row immediately after each BO iteration completes. Do not buffer.

---

## diagnostics.csv — Method-specific diagnostics

Written every K iterations (K specified in config, typically 5 or 10). One row per (iteration, seed).

| Column | Type | Description | Required |
|--------|------|-------------|----------|
| `run_id` | str | Experiment run identifier | YES |
| `seed` | int | Random seed | YES |
| `iteration` | int | BO iteration number | YES |
| `lambda_1` | float | Largest eigenvalue of G_x at x_best | METHOD_SPECIFIC |
| `lambda_2` | float | Second eigenvalue of G_x at x_best | METHOD_SPECIFIC |
| `rho` | float | λ₁/(λ₁+λ₂) — effective Fisher rank | METHOD_SPECIFIC |
| `spread_metric` | float | Mean Σᵢ∈S aᵢ²Lᵢ² over RAASP draws this iteration | METHOD_SPECIFIC |
| `subspace_angle` | float | Mean angle (°) between A(x_best) and A(x_0) | METHOD_SPECIFIC |
| `z_score` | float | (μ(x_best) - f*) / σ(x_best) | STANDARD |
| `trust_region_size` | float | TuRBO trust region max side length | STANDARD |
| `n_successes` | int | TuRBO success counter | STANDARD |
| `n_failures` | int | TuRBO failure counter | STANDARD |
| `lengthscale_mean` | float | Mean of GP ARD lengthscales | STANDARD |
| `lengthscale_min` | float | Min GP ARD lengthscale | STANDARD |
| `lengthscale_max` | float | Max GP ARD lengthscale | STANDARD |
| `mll` | float | Marginal log-likelihood of GP fit | STANDARD |
| `timestamp` | str | ISO 8601 timestamp | YES |

**Notes:**
- `METHOD_SPECIFIC` columns are required if the method computes them; otherwise set to NaN.
- `STANDARD` columns should be present for all TuRBO-based methods.
- The diagnostic write frequency must be stored in config: `"diagnostic_freq": 5`.
- If a diagnostic computation fails (e.g., G_x is singular), write NaN and log a warning — do not crash.

---

## heartbeat.json — Cluster health signal

Written every 30 seconds by `health_check.py`. Not a CSV — JSON.

```json
{
  "timestamp": "2026-03-17T14:23:01Z",
  "run_id": "{run-id}",
  "iteration": 87,
  "seeds_active": 20,
  "seeds_crashed": [],
  "regret_current": 0.234,
  "regret_stderr": 0.041,
  "regret_trend": "plateau",
  "regret_slope_last_30": 0.0001,
  "health": "yellow",
  "flags": [
    "warn_plateau: slope=0.0001 (threshold -0.001)"
  ],
  "custom_flags": [
    "check_method_losing: ratio=0.95 — within range"
  ],
  "lambda_1_last": 0.023,
  "lambda_2_last": 0.001,
  "rho_last": 0.958,
  "elapsed_minutes": 14.2,
  "timeout_minutes": 30,
  "pct_budget_used": 43.5
}
```

Fields:
- `health`: `green` | `yellow` | `red`
- `flags`: list of fired default checks with their trigger values
- `custom_flags`: list of fired custom checks (from `custom_checks.py`) with trigger values
- All values are computed from the most recent data available; NaN if not computable

---

## interrupt.json — Interrupt signal

Written by Main Agent, watcher, or human. Polled by running experiment.

```json
{
  "interrupt": false,
  "reason": "",
  "relaunch": true,
  "set_by": "main_agent | watcher | human",
  "timestamp": "2026-03-17T14:30:00Z"
}
```

- `interrupt: true` triggers a clean exit on the next poll
- `relaunch: true` means the watcher will relaunch with the updated config after exit
- `relaunch: false` means stop permanently

---

## completion_flag.json — Exit signal

Written by the experiment on exit.

```json
{
  "status": "complete | interrupted",
  "run_id": "{run-id}",
  "iterations_completed": 190,
  "seeds_completed": 20,
  "timestamp": "2026-03-17T15:01:23Z",
  "exit_reason": "budget_exhausted | interrupt_received | error"
}
```

The watcher monitors this file. When it appears, the watcher triggers the Summarizer Agent pipeline.

---

## config.json — Experiment configuration

Written by Main Agent during setup. Read by the experiment entry point.

```json
{
  "run_id": "riemannian-bo-001",
  "status": "planned | ready_to_launch | running | complete | failed",
  "hypothesis": "H001",
  "project": "riemannian-bo",

  "method_name": "SA-RAASP-TuRBO",
  "baseline_name": "Uniform-RAASP-TuRBO",
  "benchmark_name": "Hartmann6-100D-Rotated",
  "benchmark_dim": 100,
  "benchmark_rotation_seed": 42,

  "budget": 200,
  "n_seeds": 20,
  "initial_sobol": 10,
  "sequential": 190,
  "seeds": [0, 1, 2, ..., 19],

  "kernel": "SE-ARD",
  "acquisition": "LogEI",
  "n_raasp_perturbations": 20,
  "raasp_subset_size": 20,
  "interrupt_poll_every": 10,
  "diagnostic_freq": 5,

  "timeout_minutes": 60,
  "device": "cpu",

  "paths": {
    "exp_dir": "projects/riemannian-bo/experiments/riemannian-bo-001",
    "src_dir": "projects/riemannian-bo/src",
    "log": "projects/riemannian-bo/experiments/riemannian-bo-001/log.csv",
    "diagnostics": "projects/riemannian-bo/experiments/riemannian-bo-001/diagnostics.csv",
    "heartbeat": "projects/riemannian-bo/experiments/riemannian-bo-001/heartbeat.json",
    "interrupt": "projects/riemannian-bo/experiments/riemannian-bo-001/interrupt.json",
    "completion_flag": "projects/riemannian-bo/experiments/riemannian-bo-001/completion_flag.json"
  }
}
```
