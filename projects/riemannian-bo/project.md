# Project: Riemannian BO

## One-line summary

Testing whether pullback information geometry of the GP posterior map provides principled, practical improvements to high-dimensional Bayesian optimisation.

---

## Core theoretical observation

A GP with scalar output maps every input $x \in \mathbb{R}^d$ to a predictive distribution $\mathcal{N}(\mu(x), \sigma^2(x))$ on the 2-dimensional Gaussian statistical manifold. The pullback of the Fisher–Rao metric through this map defines a rank-≤2 positive semidefinite matrix $G_x$ on input space. Every acquisition function of the marginal moments $(\mu(x), \sigma(x))$ — EI, LogEI, PI, UCB — has its gradient confined to the 2D range of $G_x$. The remaining $d-2$ directions are invisible to the acquisition function.

This is the **rank-2 bottleneck**. It is the structural root cause underlying several known HDBO failure modes.

---

## Hypothesis ladder

Experiments are ordered by dependency. Run H001 before H002, etc.

| ID | Title | Status | Verdict |
|----|-------|--------|---------|
| H001 | SA-RAASP vs Uniform RAASP on Rotated Hartmann | PLANNED | — |
| H002 | NGAO vs L-BFGS for Acquisition Optimisation | PLANNED | — |
| H003 | Fisher–Rao Trust Regions vs Euclidean Boxes | PLANNED | — |
| H004 | FASM-informed SAASBO | PLANNED | — |
| H005 | Full RiemannBO vs All Baselines | PLANNED | — |

---

## Benchmarks

### Synthetic (known ground truth, known ∇f)

| Name | d | Active dims | Variant | Notes |
|------|---|-------------|---------|-------|
| Hartmann-6-100D-AxisAligned | 100 | 6 | Axis-aligned embedding | Standard HDBO benchmark |
| Hartmann-6-100D-Rotated | 100 | 6 | Rotated by Q ∈ O(100), seed=42 | Key test for non-axis-aligned methods |
| Branin-2-50D-Rotated | 50 | 2 | Rotated, seed=42 | Simpler, faster validation |
| Levy-10-200D-Embedded | 200 | 10 | Axis-aligned embedding | Large-scale test |

**Rotation convention:** Pre-multiply the input by a random orthogonal matrix Q. Q is generated once and stored as a `.npy` file in `src/benchmarks/rotations/`. All experiments use the same Q for a given (benchmark, seed) pair.

**Normalisation convention:** Regret is normalised to [0, 1] using the known optimum f* and the worst observed value after initial Sobol sampling: `regret = (f* - best_y) / (f* - y_sobol_min)`.

### Real-world

| Name | d | Source | Notes |
|------|---|--------|-------|
| Mopta08 | 124 | Vehicle crash design (Mopta Competition 2008) | Real constraint; use penalty |
| Rover Trajectory | 60 | Path planning simulation | All dimensions interact |
| LunarLander | 12 | BoTorch benchmark | Low-d sanity check |

---

## Baselines

All baselines use SE-ARD kernel and LogEI acquisition unless otherwise specified.

| Name | Description | Code location |
|------|-------------|---------------|
| TuRBO-Unif | TuRBO with uniform RAASP (Papenmeier et al. 2025 default) | `src/methods/turbo_uniform.py` |
| Vanilla-sqrt-d | Hvarfner et al. 2024 with √d-scaled lengthscale prior | `src/methods/vanilla_sqrtd.py` |
| SAASBO | Eriksson & Jankowiak 2021 fully Bayesian GP | `src/methods/saasbo.py` |
| BAxUS | Papenmeier et al. 2022 adaptive subspace | `src/methods/baxus.py` |

---

## Evaluation protocol

- **Budget:** 200 function evaluations (10 initial Sobol + 190 sequential)
- **Seeds:** 20 independent seeds per method
- **Metric:** Normalised simple regret at t ∈ {50, 100, 150, 200}
- **Statistical test:** Wilcoxon signed-rank (paired by seed, two-sided, α = 0.1)
- **Effect size:** Bootstrap 95% CI (10,000 samples, resample by seed)

---

## Library stack

| Package | Version | Purpose |
|---------|---------|---------|
| botorch | ≥ 0.10 | GP, acquisition functions, TuRBO |
| gpytorch | ≥ 1.11 | GP kernel and inference |
| torch | ≥ 2.1 | Autograd, GPU |
| numpy | ≥ 1.24 | Array ops |
| pandas | ≥ 2.0 | CSV I/O |
| scipy | ≥ 1.10 | Wilcoxon test, bootstrap |

All experiments must run on CPU (no GPU dependency) unless explicitly noted.

---

## Code conventions

- All methods are subclasses of `BaseMethod` in `src/methods/base.py`
- All benchmarks are subclasses of `BaseBenchmark` in `src/benchmarks/base.py`
- GP fitting uses `botorch.fit.fit_gpytorch_mll` with 100 L-BFGS-B steps
- Lengthscale prior: `LogNormal(log(sqrt(d)), 1)` following Hvarfner et al.
- Noise prior: `Gamma(1.1, 0.05)`
- Candidate generation: 1000 initial Sobol candidates, then RAASP perturbations
- Acquisition optimisation: L-BFGS-B with 10 random restarts (or NGAO as replacement)

---

## Known failure modes (from literature)

1. **MLL gradient starvation** (Papenmeier et al. 2025): In high dimensions, the MLL gradient w.r.t. lengthscales vanishes when lengthscales are initialised too small. Fix: initialise at √d.

2. **Acquisition flatness** (Papenmeier et al. 2025): The acquisition surface is nearly flat in $d-2$ directions. Fix: RAASP perturbations generate diverse starting points.

3. **Rank-1 collapse** (this project): When ∇μ ∥ ∇σ, $G_x$ has effective rank 1. Both eigenvalues exist but the eigenvectors span the same direction. Happens near convergence when trust region is small. Fix (proposed): Tikhonov regularisation in NGAO; monitor via ρ.

4. **Rotation sensitivity** (observed empirically): Methods that assume axis-aligned active subspaces (SAASBO, RAASP with coordinate perturbations) fail on rotated problems. Fix (proposed): SA-RAASP, FRTR.

---

## Paper targets

Primary venue: NeurIPS 2026  
Submission deadline: May 2026  
Current draft: `../../neurips_paper_v2.tex` (gradient bottleneck framing, H001–H004 results needed)  
Extended draft: `../../riemannian_acqf_optimization.tex` (Riemannian methods framing, H001–H005 results needed)
