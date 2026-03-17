# H001: SA-RAASP vs Uniform RAASP on Rotated Hartmann-6/100D

## Claim

Perturbing TuRBO starting points along the top-2 eigenvectors of the pullback Fisher metric $G_x$ (SA-RAASP) generates more informative acquisition optimisation starting points and achieves lower normalised regret than TuRBO with uniform axis-aligned RAASP, specifically on benchmarks where the active subspace is not axis-aligned.

---

## Mechanism

### The bottleneck that motivates SA-RAASP

A GP with scalar output defines a smooth map $\varphi: \mathbb{R}^d \to \mathcal{M}^2$ via $x \mapsto (\mu(x), \sigma(x))$. The pullback of the Fisher–Rao metric through $\varphi$ gives:

$$G_x = \frac{1}{\sigma^2} \nabla\mu \nabla\mu^\top + \frac{2}{\sigma^2} \nabla\sigma \nabla\sigma^\top \in \mathbb{R}^{d \times d}$$

Since this is a sum of two rank-1 matrices, $\text{rank}(G_x) \leq 2$. Every acquisition function $\alpha(x) = h(\mu(x), \sigma(x))$ has its gradient $\nabla_x \alpha \in \mathcal{A}(x) = \text{span}\{\nabla\mu, \nabla\sigma\}$, a 2D subspace of $\mathbb{R}^d$. The remaining $d-2$ directions form the null space of $G_x$ and are invisible to the acquisition function.

### Why uniform RAASP fails on rotated problems

RAASP (Papenmeier et al. 2025) generates starting points by perturbing $x^*$ along a random subset $S$ of coordinate axes:

$$x_0 = x^* + \varepsilon \odot \mathbf{1}_S, \quad \varepsilon_i \sim \mathcal{N}(0, L_i^2) \text{ for } i \in S$$

The purpose of RAASP is to generate diverse starting points for the multi-start L-BFGS acquisition optimiser, not to restrict which dimensions L-BFGS moves in. L-BFGS always runs in all $d$ dimensions.

The key quantity is the first-order acquisition change induced by the perturbation:

$$\text{acqf}(x_0) - \text{acqf}(x^*) \approx \sum_{i \in S} a_i \varepsilon_i$$

where $a_i = [\nabla_x \alpha(x^*)]_i$. Since $\nabla_x \alpha \in \mathcal{A}(x^*)$, the components $a_i$ are large only for dimensions $i$ where the coordinate axis $e_i$ has significant overlap with the active subspace $\mathcal{A}(x^*)$.

**On axis-aligned problems:** $\mathcal{A}(x^*)$ tends to align with coordinate axes (the GP quickly learns which coordinates are active). Even uniform RAASP will occasionally select dimensions with large $a_i$.

**On rotated problems:** $\mathcal{A}(x^*)$ is a rotated subspace. The coordinate axes $e_i$ all have roughly equal, small overlap with $\mathcal{A}(x^*)$ (overlap ≈ $1/\sqrt{d}$ for a uniformly random subspace). Uniform RAASP generates starting points with near-zero acquisition change from $x^*$ — they are essentially all in the same basin of attraction as $x^*$, and L-BFGS finds the same local optimum from each.

This is the **RAASP null-space problem**: on rotated benchmarks, most random coordinate perturbations fall in $\ker(G_x)$ and produce no useful starting-point diversity.

### Why SA-RAASP fixes this

SA-RAASP perturbs along the eigenvectors of $G_x$:

$$x_0 = x^* + \sum_{j=1}^{s} \varepsilon_j \mathbf{v}_j, \quad \varepsilon_j \sim \mathcal{N}(0, L_j^2)$$

where $\mathbf{v}_1, \mathbf{v}_2$ are the top-2 eigenvectors of $G_x$ (spanning $\mathcal{A}(x^*)$) and $L_j = \Delta / \sqrt{\lambda_j + \delta}$ is the Fisher–Rao trust region semi-axis length.

Since $\nabla_x \alpha \in \mathcal{A}(x^*) = \text{span}\{\mathbf{v}_1, \mathbf{v}_2\}$, the acquisition change from an SA-RAASP perturbation is:

$$\text{acqf}(x_0) - \text{acqf}(x^*) \approx a_1 \varepsilon_1 + a_2 \varepsilon_2$$

with expected squared change $a_1^2 L_1^2 P[1 \in S] + a_2^2 L_2^2 P[2 \in S]$.

SA-RAASP sets $P[j \in S] \propto \lambda_j$, maximising this expected squared change. The remaining $s - 2$ perturbation dimensions are drawn from a uniform distribution over the null space, maintaining exploration.

**On rotated problems:** SA-RAASP always hits both informative directions, regardless of how the active subspace is oriented. Starting-point spread should be $\sim d/2$ times higher than uniform RAASP (since uniform RAASP has probability $\approx 2/d$ of selecting the two informative directions).

**On axis-aligned problems:** SA-RAASP and uniform RAASP are equivalent in expectation once the GP has correctly identified the active dimensions (both select the active coordinate axes). The advantage of SA-RAASP is small.

---

## Prediction

SA-RAASP achieves **≥ 15% lower normalised regret at t = 200** compared to TuRBO with uniform RAASP on Hartmann-6 embedded in d = 100 with a random rotation matrix (seed = 42), averaged over 20 seeds (mean ± SE).

Secondary prediction: SA-RAASP mean starting-point spread $\sum_{i \in S} a_i^2 L_i^2$ is **≥ 3× higher** than uniform RAASP on the rotated benchmark.

---

## Falsification

The hypothesis is **rejected** if:

- Effect size < 5% improvement OR p > 0.1 (Wilcoxon signed-rank, paired by seed, n = 20), OR
- The spread metric is NOT significantly higher for SA-RAASP (< 2× or p > 0.1 on spread ratio)

The hypothesis is **inconclusive** if:

- Effect size is in [5%, 15%) — directionally correct but below the predicted threshold; run with more seeds or a different benchmark
- The experiment was killed before 80% of budget due to a non-method-related failure

---

## Experiment Plan

```yaml
treatment:
  method: SA-RAASP-TuRBO
  description: >
    TuRBO with SA-RAASP candidate generation. Perturbation directions are the
    top-2 eigenvectors of G_x = (1/σ²)∇μ∇μᵀ + (2/σ²)∇σ∇σᵀ, computed at
    x_best via two autograd backward passes. Remaining s-2 perturbation 
    dimensions sampled uniformly from coordinate axes (exploration floor).
    Step sizes: L_j = Δ / sqrt(λ_j + δ), δ = 1e-4.

control:
  method: Uniform-RAASP-TuRBO
  description: >
    Standard TuRBO implementation from BoTorch with RAASP as described in
    Papenmeier et al. (ICML 2025). s dimensions chosen uniformly at random
    from {1,...,d} without replacement.

benchmark:
  name: Hartmann6-100D-Rotated
  function: Hartmann-6
  dimension: 100
  active_dimensions: 6
  embedding: random_rotation
  rotation_seed: 42
  rotation_file: src/benchmarks/rotations/hartmann6_100d_rot42.npy

budget:
  evaluations_per_seed: 200
  seeds: 20
  initial_sobol: 10
  sequential: 190
  seeds_list: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19]

kernel: SE-ARD
acquisition: LogEI
n_raasp_perturbations: 20
raasp_subset_size: 20
sa_raasp_n_eigenvector_dims: 2
sa_raasp_exploration_floor_factor: 1.0  # uniform over remaining dims
tikhonov_delta: 1.0e-4

interrupt_poll_every: 10
diagnostic_freq: 5
timeout_minutes: 90
```

---

## Diagnostics to Track

All diagnostics written to `diagnostics.csv` every 5 iterations per seed.

### Primary diagnostics (required for verdict)
- `regret`: normalised simple regret at each iteration
- `best_y`: best observed function value

### Method-specific (SA-RAASP)
- `spread_metric_method`: $\sum_{i \in S} a_i^2 L_i^2$ per RAASP draw, method (mean over all draws this iteration)
- `spread_metric_baseline`: same for uniform RAASP baseline
- `spread_ratio`: `spread_metric_method / spread_metric_baseline`
- `subspace_angle`: angle in degrees between $\mathcal{A}(x^*)$ and the perturbation subspace spanned by the selected dimensions
- `n_active_dims_hit`: number of the 6 active (rotated) dimensions whose projection on the perturbation subspace exceeds 0.1

### Spectral (key for theory check)
- `lambda_1`: largest eigenvalue of $G_x$ at $x^*$
- `lambda_2`: second eigenvalue of $G_x$ at $x^*$
- `rho`: $\lambda_1 / (\lambda_1 + \lambda_2)$ — effective Fisher rank; closer to 0.5 is healthier
- `lambda_1_eigvec_active_overlap`: dot product of $\mathbf{v}_1$ with the known active subspace (available because this is a synthetic benchmark with known Q)

### Standard TuRBO diagnostics
- `trust_region_size`: max side length of current TR
- `n_successes`: TuRBO success counter
- `n_failures`: TuRBO failure counter
- `lengthscale_mean`, `lengthscale_min`, `lengthscale_max`: GP ARD lengthscales
- `mll`: marginal log-likelihood at fitted parameters
- `z_score`: $(μ(x^*) - f^*) / σ(x^*)$ — exploitation/exploration indicator

### For correlation analysis
- `acqf_optima_value`: acquisition function value at the best starting point found by L-BFGS
- `n_lbfgs_restarts`: number of L-BFGS restarts that converged
- `lbfgs_spread`: std of acquisition values across L-BFGS starting points (measure of starting point diversity)

---

## Stopping Rules

```yaml
stopping_rules:

  early_stop_method_clearly_losing:
    description: >
      Stop if SA-RAASP is clearly worse than uniform RAASP past the midpoint.
      This is a directional test — if the method is moving in the wrong direction,
      stop early and investigate rather than wasting budget.
    trigger: "iteration > 100 AND mean(regret_method_last_20) > mean(regret_baseline_last_20) * 1.25"
    severity: red
    window: 20
    note: >
      The 1.25 threshold gives SA-RAASP room to be slightly worse early (it may
      take time to estimate G_x eigenvectors accurately). Only fires if it's
      clearly and persistently losing by > 25%.

  early_stop_nan:
    description: "Any NaN in regret or diagnostics"
    trigger: "any(isnan(regret))"
    severity: red
    window: 1

  warn_rank1_collapse:
    description: >
      ρ approaching 1 means G_x is rank-1: ∇μ ∥ ∇σ. SA-RAASP cannot distinguish
      two informative directions. Spread should degrade. This is a theory-relevant
      event — we want to know when it happens.
    trigger: "mean(rho_last_10) > 0.90"
    severity: yellow
    window: 10
    note: >
      This is WARNING not RED because rank-1 collapse is expected near convergence.
      We want to log when it happens, not stop the experiment.

  warn_spread_ratio_degrading:
    description: >
      If SA-RAASP spread advantage over uniform RAASP is declining consistently,
      the mechanism may be breaking down. Flag for monitor check-in.
    trigger: "mean(spread_ratio_last_20) < 1.5 AND iteration > 50"
    severity: yellow
    window: 20
    note: >
      Below 1.5× ratio, SA-RAASP is barely outperforming uniform RAASP in
      starting-point diversity. The effect on regret may be too small to detect.

  warn_lbfgs_convergence_poor:
    description: >
      If most L-BFGS restarts are finding the same acquisition optimum (low spread
      across starting points), the diverse starting points are not being exploited.
    trigger: "mean(lbfgs_spread_last_10) < 0.001"
    severity: yellow
    window: 10

  warn_trust_region_collapsed:
    description: >
      If the trust region has collapsed to side length < 1e-5, TuRBO is in
      its terminal phase. Further iterations won't improve regret much.
      This is informational — do not stop the experiment.
    trigger: "mean(trust_region_size_last_5) < 1e-5"
    severity: yellow
    window: 5
    note: "This is expected behaviour for TuRBO near convergence — just log it."

  error_g_x_singular:
    description: >
      If G_x eigenvectors cannot be computed (zero matrix), the SA-RAASP
      implementation is falling back to uniform RAASP. Flag as error.
    trigger: "any(isnan(lambda_1))"
    severity: red
    window: 1
    note: >
      NaN in lambda_1 means G_x computation failed. This is a code bug, not
      a method failure. Interrupt and fix.
```

---

## Statistical Analysis Plan

```yaml
primary_test:
  metric: normalised_regret
  aggregation: mean_over_seeds
  evaluation_point: t=200
  test: wilcoxon_signed_rank
  paired: true
  alpha: 0.1
  effect_size_threshold_pct: 15.0

secondary_tests:
  - name: regret_at_t50_t100_t150
    metric: normalised_regret
    evaluation_points: [50, 100, 150]
    test: wilcoxon_signed_rank
    note: "To characterise early vs late dynamics"

  - name: spread_ratio_test
    metric: spread_ratio
    aggregation: mean_over_seeds_over_iterations
    test: wilcoxon_signed_rank
    note: "Tests the mechanism — is spread actually higher?"

  - name: rho_vs_regret_correlation
    metric: rho_at_t20_vs_final_regret
    test: spearman_rank_correlation
    note: "Tests spectral diagnostic as convergence predictor"

  - name: active_dims_hit_rate
    metric: n_active_dims_hit
    aggregation: mean_over_all_draws_all_seeds
    test: t_test
    note: "Does SA-RAASP actually find the active dimensions more often?"

bootstrap:
  n_samples: 10000
  resample_unit: seed
  confidence_level: 0.95
  metric: effect_size_pct

multiple_comparison_correction: bonferroni
n_tests: 5  # primary + 4 secondary
adjusted_alpha: 0.02  # 0.1 / 5
```

---

## Relevant Theory

### Supporting theorems

**Theorem 3.1 (Pullback Degeneracy)** from `riemannian_acqf_optimization.tex`:  
→ Establishes that $\nabla_x \alpha \in \mathcal{A}(x)$, the 2D active subspace.  
→ This is the foundation of the mechanism. If this theorem is wrong, the whole argument fails.  
→ Status: Elementary — follows from chain rule. No gap.

**Theorem 5.1 (SA-RAASP Spread Dominance)** from `riemannian_acqf_optimization.tex`:  
→ Shows that $P[j \in S] \propto \lambda_j$ maximises expected acquisition spread $\sum_j (a \cdot v_j)^2 L_j^2 P[j \in S]$.  
→ Used because: $\nabla_x \alpha \in \text{span}\{v_1, v_2\}$, so only $j \in \{1,2\}$ contribute.  
→ **Gap**: The theorem is first-order only. The O(||δ||²) correction from the non-flat acquisition surface could in principle reduce spread if the acquisition surface is highly curved. However, this correction is O(||δ||²/σ), and since the trust region is small near $x^*$, ||δ|| is small and the correction is negligible.  
→ **Gap**: The theorem assumes $a \in \text{span}\{v_1, v_2\}$ exactly. In practice, the eigenvectors of $G_x$ are approximated via autograd (exact, no approximation error) but are computed at $x^*$ while perturbations move away from $x^*$. The subspace rotates as we move, but for small perturbations (trust region) this rotation is O(||δ||).

**Theorem 2.1 (Alignment Bound)** from `neurips_paper_v2.tex`:  
→ Shows $\cos(\nabla_x \alpha, \nabla f) \leq \sqrt{\alpha_t}$ where $\alpha_t$ is the alignment score.  
→ Relevant because: if $G_x$ eigenvectors align with $\nabla f$ (true on the rotated benchmark where the GP eventually finds the right subspace), SA-RAASP's perturbations are more informative.  
→ **Gap**: This theorem is about the alignment of the acquisition gradient with the objective gradient, not about starting-point diversity. It motivates but does not directly prove the SA-RAASP advantage.

### Known gaps between theory and experiment

1. **First-order vs full nonlinear effect**: All theorems are first-order in the perturbation size. The actual effect on starting-point diversity and final regret is the integral of a nonlinear function. Theory says the direction is right; magnitude may differ.

2. **Eigenvector accuracy near convergence**: As the trust region shrinks, $G_x$ becomes more ill-conditioned (both λ values shrink). The eigenvectors become less meaningful. SA-RAASP may degrade to near-uniform RAASP in the terminal phase. The `warn_rank1_collapse` stopping rule monitors this.

3. **Rotation coverage**: With $s = 20$ perturbation dimensions and only 2 informative directions, SA-RAASP generates 2 "informative" starting points and 18 "noise" starting points per iteration. Uniform RAASP generates 20 random starting points. The diversity advantage is concentrated in the 2 eigenvector directions.

---

## Intuitions and Motivations (Extended)

*These are the informal intuitions that motivate the hypothesis. They are not rigorous. They exist to help the Main Agent understand the spirit of the experiment.*

### Intuition 1: The null-space problem is severe on rotated benchmarks

On a 100D rotated problem with 6 active dimensions, the informative subspace $\mathcal{A}(x^*)$ is a 2D subspace of a 100D space oriented in a generic (non-axis-aligned) direction. Uniform RAASP picks 20 coordinate axes at random. The probability that any of these 20 axes has significant overlap with $\mathcal{A}(x^*)$ is roughly $20 \times 2/100 = 0.4$, which means on average 8/20 starting points have some (small) acquisition variation, and 12/20 are near-identical to $x^*$. SA-RAASP guarantees that 2/20 starting points are in exactly the right directions — and these 2 are likely to find better acquisition optima than the 8 random successes from uniform RAASP.

### Intuition 2: The eigenvectors of G_x should track the active subspace

Even if the GP's lengthscales haven't perfectly converged, the eigenvectors of $G_x$ should point toward the directions where the GP's predictions change most. On a rotated 6D active problem, after 50–100 observations, the GP should have enough data to identify that predictions change along the rotated active dimensions. These should appear as the large-eigenvalue eigenvectors of $G_x$.

This intuition is partially circular: if the GP is well-calibrated, SA-RAASP works. If the GP is poorly calibrated (lengthscale collapse), both $G_x$ eigenvectors degenerate and SA-RAASP falls back to random. The `warn_rank1_collapse` and `error_g_x_singular` stopping rules monitor this.

### Intuition 3: L-BFGS is the bottleneck, not the acquisition function

On rotated problems, the issue is not that the acquisition function is badly designed — it is that L-BFGS starts from points with nearly identical acquisition values (because of the null-space problem). L-BFGS is excellent at fine-tuning a starting point; it is poor at global exploration. SA-RAASP provides the diversity that L-BFGS cannot generate on its own.

This means SA-RAASP is a complement to L-BFGS, not a replacement. The combined system (SA-RAASP starting points + L-BFGS refinement) should outperform either alone.

### Intuition 4: The effect should be large on rotated and small on axis-aligned

If the active subspace is aligned with coordinate axes, uniform RAASP already hits the active dimensions with reasonable frequency (e.g., on a 6D active / 100D total problem with 20 RAASP dimensions, probability $\approx 1 - \binom{94}{20}/\binom{100}{20} \approx 0.73$ of hitting at least one active dim). SA-RAASP's advantage is small. On rotated problems, the coordinate axes are uninformative and the advantage is large.

This is a strong prediction that we can verify: H001 on rotated benchmark should show large effect; if we run H001 on axis-aligned benchmark as a control, effect should be small. We have included this as a sanity check in the secondary analysis.

### Intuition 5: The spread metric is the mechanistic test

If the hypothesis is correct, SA-RAASP should have higher spread metric than uniform RAASP. If SA-RAASP achieves lower regret but NOT higher spread metric, the mechanism is wrong — the improvement is coming from somewhere else (perhaps the eigenvector perturbation accidentally improves the acquisition value in a way we don't understand).

If SA-RAASP achieves higher spread metric but NOT lower regret, the starting-point diversity is not translating to better acquisition optima — perhaps L-BFGS is failing to exploit the diverse starting points.

Both of these would be informative failures.

---

## Dependencies

None — this is the first experiment.

---

## Expected Difficulty

**MEDIUM**

- The implementation is straightforward (two autograd backward passes + eigendecomposition of a 2×2 matrix)
- The benchmark is well-understood (Hartmann-6 is standard; random rotation is deterministic given seed)
- The main uncertainty is whether the eigenvectors of $G_x$ stabilise quickly enough on a rotated problem with 20 seeds × 200 evaluations

---

## Open Questions

1. **Eigenvector stability:** How quickly do the eigenvectors of $G_x$ at $x^*$ stabilise during a BO run on the rotated benchmark? If they are noisy for the first 50 iterations (because the GP hasn't learned the active subspace yet), the SA-RAASP advantage may be delayed.

2. **Interaction with TuRBO restarts:** TuRBO resets the trust region when it shrinks too small. After a reset, $G_x$ may change direction suddenly. How does SA-RAASP behave across restarts?

3. **Rotation seed sensitivity:** Is seed=42 a "nice" rotation (one that is genuinely non-axis-aligned and thus hard for uniform RAASP) or a degenerate one? We should verify this by checking the mutual incoherence $\|Q_{6\times 100}\|_\infty$ — if any column of Q has a large entry in one of the 6 active positions, the rotation is near-axis-aligned.

4. **What if ρ collapses early?** If rank-1 collapse (ρ → 1) happens before iteration 100, SA-RAASP cannot distinguish the two eigenvectors meaningfully. Should we fall back to Fisher-diagonal weighting (FI-RAASP from the `neurips_paper_v2.tex` paper) in this case?
