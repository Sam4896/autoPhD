# Prior Work Summary

Key references for the Riemannian BO project. Entries are sorted by relevance to the current hypothesis ladder.

---

## Directly relevant to H001–H005

### Papenmeier et al. (ICML 2025) — Understanding High-Dimensional BO

**Key findings:**
- RAASP (Random Axis-Aligned Subspace Perturbation) is the critical component of TuRBO's HDBO success, not the trust region geometry
- MLL gradient starvation (not acquisition gradient vanishing) is the primary obstacle to GP fitting in high dimensions
- Uniform RAASP generates diverse starting points that allow multi-start L-BFGS to find different acquisition optima
- Without RAASP, TuRBO collapses to a local search that cannot escape basins of attraction

**Relevance to our work:**
- Confirms that starting-point diversity (via RAASP) is the mechanism — this is exactly what H001 improves
- The "MLL gradient starvation" finding is complementary to our "acquisition gradient bottleneck" finding — they are different problems (GP fitting vs acquisition optimisation)
- Our SA-RAASP is a principled replacement for their uniform RAASP

---

### Hvarfner, Hellsten, Nardi (ICML 2024) — Vanilla BO Performs Great in High Dimensions

**Key findings:**
- Initialising lengthscales at $c\sqrt{d}$ (with LogNormal prior) resolves MLL gradient starvation
- Vanilla BO (no special structure) with this prior is competitive with TuRBO and SAASBO on many benchmarks
- LogEI (Ament et al. 2024) is necessary for numerical stability

**Relevance to our work:**
- Sets the baseline for "vanilla" high-dimensional BO — we compare against this
- The $\sqrt{d}$ initialisation is a prerequisite for our methods to work (lengthscale collapse → $G_x$ collapse)
- All our experiments use the LogNormal($\log\sqrt{d}$, 1) prior following this paper

---

### Eriksson et al. (NeurIPS 2019) — TuRBO

**Key findings:**
- Local trust region BO outperforms global BO in high dimensions
- The trust region starts large and contracts geometrically on failures; expands on successes
- Multiple interleaved trust regions (TuRBO-m) perform similarly to single (TuRBO-1) for most problems

**Relevance to our work:**
- TuRBO is our primary baseline and the host algorithm for SA-RAASP (H001) and FRTR (H003)
- Trust region geometry is a key part of our Fisher–Rao trust region (H003)
- Our FRTR replaces the Euclidean hyperrectangle with a pullback-metric ellipsoid

---

### Ament et al. (NeurIPS 2023 Spotlight) — LogEI

**Key findings:**
- Standard EI underflows to machine zero when the standardised improvement $z \ll 0$
- LogEI = $\log(\text{EI})$ computed stably via log-sum-exp avoids underflow
- LogEI recovers nonzero gradients where EI gives machine-zero gradients
- This is a numerical fix, NOT a structural fix — the gradients still lie in $\mathcal{A}(x)$

**Relevance to our work:**
- We use LogEI in all experiments (follows Hvarfner et al.)
- The distinction between LogEI's numerical fix and our structural fix is a key contribution of our theory section
- Remark 3.1 in `riemannian_acqf_optimization.tex` explicitly clarifies this

---

## Information geometry and natural gradient

### Amari (2016) — Information Geometry and Its Applications

**Key results:**
- Fisher information metric as the canonical Riemannian metric on statistical manifolds
- Dually flat structure of exponential families: natural and expectation parameters
- Natural gradient = steepest ascent on the statistical manifold

**Relevance:**
- Provides the theoretical foundation for the pullback metric $G_x$
- The Gaussian manifold $\mathcal{M}^2$ is the specific manifold we use
- Chapter 2 (exponential families) and Chapter 3 (Fisher metric) are essential reading

---

### Ollivier et al. (JMLR 2017) — Information-Geometric Optimization

**Key results:**
- CMA-ES is natural gradient ascent on the Gaussian manifold (parameterising the search distribution)
- IGO flow achieves maximal invariance under reparameterisation
- Recovers CMA-ES, PBIL, and other evolutionary algorithms as special cases

**Relevance:**
- Our NGAO (H002) is structurally similar to IGO but operates on the acquisition function, not the search distribution
- The natural gradient optimality argument in Theorem 4.1 of our paper is analogous to Ollivier's results

---

### Arvanitidis et al. (AISTATS 2022) — Pulling Back Information Geometry

**Key results:**
- Pullback of Fisher–Rao metric through a smooth map defines a degenerate Riemannian metric
- Geodesics of the pullback metric minimise KL divergence between successive distributions along the path
- Framework applies to any differentiable map from parameter space to a statistical manifold

**Relevance:**
- Our pullback metric $G_x = J^\top G_\theta J$ is a direct instance of their framework
- We cite this as the rigorous mathematical foundation for the pullback degeneracy

---

## Active subspace methods

### Constantine, Dow, Wang (SIAM J. Sci. Comput. 2014) — Active Subspace Methods

**Key results:**
- The active subspace of a function $f$ is defined by the eigendecomposition of $C = \mathbb{E}_\rho[(\nabla f)(\nabla f)^\top]$
- The top eigenvectors of $C$ identify directions of maximum average gradient magnitude
- Low-rank approximation of $C$ enables dimension reduction for expensive simulations

**Relevance:**
- Our FASM (Fisher Active Subspace Matrix) is a confidence-weighted, GP-posterior variant of $C$
- FASM uses $\nabla_x \mu$ instead of $\nabla f$ (unavailable in BO) and weights by $1/\sigma^2$
- Theorem 5.1 (FASM eigenvalue dominance) shows FASM's eigenvalues dominate Constantine's exploitation-only component

---

## Riemannian optimisation

### Absil, Mahony, Sepulchre (2008) — Optimization Algorithms on Matrix Manifolds

**Key results:**
- Riemannian gradient descent and Newton's method on smooth manifolds
- Trust region methods on Riemannian manifolds (Absil, Baker, Gallivan 2007)
- Convergence guarantees under Riemannian regularity conditions

**Relevance:**
- Provides theoretical framework for NGAO (H002) and FRTR (H003)
- FRTR adapts their trust-region framework to the pullback metric setting

---

## Known null results / negative findings (important to know before experimenting)

### Rotation sensitivity is not always bad

Wang et al. (JAIR 2016) — REMBO uses random embeddings and shows that even random low-dimensional projections can work if the effective dimension is low enough. On our rotated benchmark, the 6 active dimensions are NOT a low-dimensional subspace of $[0,1]^{100}$ in a way that REMBO exploits — REMBO assumes a truly low-dimensional input space, not just a rotated one.

**Implication:** If H001 shows only modest improvement on rotated benchmarks, it may be because TuRBO is already implicitly adapting its lengthscales to the rotated subspace, making the RAASP improvement less necessary.

### SAASBO's axis-aligned assumption

SAASBO (Eriksson & Jankowiak, UAI 2021) assumes that the function has axis-aligned sparsity — some lengthscales are very long (inactive) and others are short (active). On rotated benchmarks, ALL lengthscales are moderate (the rotation distributes importance equally across coordinate axes). SAASBO fails on rotated problems. We use this as motivation for why rotation-aware methods (SA-RAASP, FRTR) are needed.

**Implication:** If our rotated benchmark shows SAASBO performing poorly, that is expected and consistent with prior work. The interesting comparison is SA-RAASP vs TuRBO-Uniform on the rotated benchmark.
