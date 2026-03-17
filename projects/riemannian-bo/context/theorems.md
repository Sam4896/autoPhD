# Theorems Reference

Key theoretical results the Main Agent can consult without re-reading the full paper drafts.  
Each entry states the theorem, its proof sketch, its scope, and its known gaps.

---

## From `neurips_paper_v2.tex` — The Gradient Bottleneck Paper

### Theorem 2.1 — Alignment Bounds Acquisition Gradient Utility

**Statement:**  
For any acquisition function $\alpha(x) = h(\mu(x), \sigma(x))$ with $\nabla_{(\mu,\sigma)} h \neq 0$:

$$\frac{|\nabla_x \alpha(x_t) \cdot \nabla f(x_t)|}{\|\nabla_x \alpha(x_t)\| \|\nabla f(x_t)\|} \leq \sqrt{\alpha_t}$$

where $\alpha_t = \|\Pi_{\mathcal{A}(x_t)} \nabla f(x_t)\|^2 / \|\nabla f(x_t)\|^2$ is the alignment score.

**Proof sketch:**  
$\nabla_x \alpha \in \mathcal{A}(x_t)$ by chain rule. Decompose $\nabla f = \Pi_\mathcal{A} \nabla f + \Pi_{\mathcal{A}^\perp} \nabla f$. Since $\nabla_x \alpha \perp \Pi_{\mathcal{A}^\perp} \nabla f$, the inner product is at most $\|\nabla_x \alpha\| \|\Pi_\mathcal{A} \nabla f\| = \|\nabla_x \alpha\| \sqrt{\alpha_t} \|\nabla f\|$.

**Scope:**  
Applies to EI, LogEI, PI, UCB, and any $h(\mu, \sigma)$. Does NOT apply to Thompson Sampling or Knowledge Gradient.

**Gap:**  
The bound is tight (achievable when $\nabla_x \alpha \parallel \Pi_\mathcal{A} \nabla f$) but says nothing about whether acquisition optimisation finds a good $x$ — only about the gradient direction.

---

### Theorem 2.2 — FI-RAASP Component Bound

**Statement:**  
For any acquisition function $h(\mu, \sigma)$, the per-dimension acquisition gradient satisfies:

$$a_i^2 \leq c \cdot g_{ii}$$

where $a_i = [\nabla_x \alpha(x)]_i$, $g_{ii} = \sigma^{-2}(\partial_i \mu)^2 + 2\sigma^{-2}(\partial_i \sigma)^2$ is the Fisher diagonal, and $c = \sigma^2 h_\mu^2 + (\sigma^2/2) h_\sigma^2$ is independent of $i$.

**Proof sketch:**  
Cauchy–Schwarz in the Fisher inner product: $a_i^2 = \langle (\partial_i \mu, \partial_i \sigma), (h_\mu, h_\sigma) \rangle^2 \leq \|(∂_i\mu, \partial_i\sigma)\|^2_{G_\theta} \cdot \|(h_\mu, h_\sigma)\|^2_{G_\theta^{-1}} = g_{ii} \cdot c$.

**Scope:**  
Applies to all marginal-moment acquisition functions. Independent of benchmark and GP configuration.

**Gap:**  
This is an upper bound, not an equality. In the exploitation-dominated regime ($|h_\mu| \gg |h_\sigma|$), the bound is tight. In exploration-dominated regimes, $g_{ii}$ may over-count due to the $(\partial_i \sigma)^2$ term. FI-RAASP's advantage is reduced in the exploration regime.

---

### Theorem 5.1 — FASM Eigenvalue Dominance

**Statement:**  
Let $\bar{\mathcal{F}} = \mathbb{E}_\rho[G_x] = \bar{\mathcal{F}}_\mu + \bar{\mathcal{F}}_\sigma$. Then $\nu_k(\bar{\mathcal{F}}) \geq \nu_k(\bar{\mathcal{F}}_\mu)$ for all $k$.

**Proof sketch:**  
$\bar{\mathcal{F}}_\sigma \succeq 0$. Weyl's inequality: $\nu_k(A+B) \geq \nu_k(A) + \lambda_{\min}(B) \geq \nu_k(A)$.

**Scope:**  
Unconditional. Holds for any GP, any training set, any benchmark.

**Gap:**  
Proves eigenvalue dominance but NOT spectral gap amplification. The gap $\nu_2 - \nu_3$ may shrink if $\bar{\mathcal{F}}_\sigma$ spreads energy across many directions. The previous version of the paper incorrectly claimed gap amplification; this was retracted.

---

## From `riemannian_acqf_optimization.tex` — The Riemannian Methods Paper

### Theorem 3.1 — Pullback Degeneracy

**Statement:**  
Let $\varphi: \mathbb{R}^d \to \mathcal{M}^2$ be the GP posterior map with $\text{rank}(J(x)) = 2$. Then:
1. $\nabla_x(h \circ \varphi) \in \mathcal{A}(x)$, a 2D subspace
2. For every $v \in \ker(J(x))$: $\nabla_x(h \circ \varphi) \cdot v = 0$
3. For Lebesgue-a.e. $g \in \mathbb{R}^d$ with $d \geq 3$: $g \notin \mathcal{A}(x)$

**Proof sketch:**  
(1) Chain rule: $\nabla_x(h \circ \varphi) = J^\top \nabla_{(\mu,\sigma)} h \in \text{col}(J^\top) = \mathcal{A}(x)$.  
(2) $v \in \ker(J)$ means $Jv = 0$, so $(J^\top \nabla h) \cdot v = (\nabla h)^\top Jv = 0$.  
(3) $\mathcal{A}(x)$ is 2D, has measure zero in $\mathbb{R}^d$ for $d \geq 3$.

**Scope:**  
All marginal-moment acquisitions. Elementary linear algebra.

**Gap:**  
None — this is a consequence of the chain rule and linear algebra.

---

### Theorem 4.1 — Optimality of the Natural Gradient

**Statement:**  
Among all $v \in \mathbb{R}^d$ with $v^\top G_x v \leq 1$, the natural gradient $\tilde{\nabla}\alpha = G_x^+ \nabla\alpha$ achieves:

$$\nabla\alpha^\top v \leq \nabla\alpha^\top G_x^+ \nabla\alpha = \|\tilde{\nabla}\alpha\|_{G_x}^2$$

Moreover, $\tilde{\nabla}\alpha \in \mathcal{A}(x)$ — the natural gradient lies in the 2D informative subspace.

**Proof sketch:**  
Cauchy–Schwarz in the $G_x$ inner product. Since $\nabla\alpha \in \mathcal{A}(x)$ and $v_\perp \in \ker(G_x)$ contributes nothing to the inner product, only the $\mathcal{A}(x)$ component of $v$ matters.

**Scope:**  
Applies to all marginal-moment acquisitions. Requires $G_x^+$ (pseudoinverse) to exist, which it does if at least one eigenvalue is nonzero.

**Gap:**  
This is a local result — it describes the steepest ascent direction at the current point, not whether following this direction globally finds the acquisition optimum. The acquisition surface may be multimodal.

---

### Theorem 5.1 — SA-RAASP Spread Dominance

**Statement:**  
Since $\nabla_x \alpha \in \mathcal{A}(x^*) = \text{span}\{v_1, v_2\}$ (the top-2 eigenvectors of $G_x$):

$$\mathbb{E}_{\text{SA}}\left[\left(a^\top (x_0 - x^*)\right)^2\right] = a_1^2 L_1^2 P[1 \in S] + a_2^2 L_2^2 P[2 \in S]$$

where $a_j = a \cdot v_j$. With $P[j \in S] \propto \lambda_j$, this is maximised over all selection distributions satisfying $\sum_j P[j \in S] = s$.

When the active subspace is not axis-aligned, $P^{\text{SA}}_j \gg P^{\text{Uniform}}_j$ for $j \in \{1,2\}$, and the spread ratio is approximately $d \lambda_j / (2(\lambda_1 + \lambda_2))$.

**Proof sketch:**  
Since $a \in \text{span}\{v_1, v_2\}$ and $v_j$ are orthonormal, $a^\top v_j = a_j \delta_{j \leq 2}$. Rearrangement inequality: to maximise $\sum_j a_j^2 P_j$ over $P$ with $\sum P_j = s$, set $P_j$ proportional to $a_j^2$. The bound $a_j^2 \leq c \cdot g_{ii}$ (Theorem 2.2) replaces $a_j^2$ with the Fisher diagonal.

**Gap:**  
1. **First-order only**: the spread formula ignores O(||δ||²) curvature corrections
2. **Assumes exact eigenvectors**: in practice, eigenvectors are computed approximately at $x^*$; for large perturbations, the true eigenvectors at $x_0$ may differ
3. **Does not account for L-BFGS behaviour**: higher spread does not automatically mean better L-BFGS optima — it means starting points are in different basins, which is necessary but not sufficient

---

## Relationship between theorems

```
Theorem 3.1 (Chain Rule, Pullback Degeneracy)
    ↓ establishes: ∇α ∈ A(x), the 2D subspace
    ↓
Theorem 4.1 (Natural Gradient Optimality)
    ← "steepest ascent on pullback manifold = G_x^+ ∇α"
    
Theorem 3.1
    ↓ establishes: a_j = a·v_j = 0 for j ≥ 3
    ↓
Theorem 5.1 (SA-RAASP Spread)
    ← "perturbing along v_1, v_2 maximises expected acquisition change"
    
Theorem 2.2 (FI-RAASP Component Bound)
    ← "Fisher diagonal upper-bounds per-dimension acquisition gradient"
    ← used as a proxy for a_j^2 in Theorem 5.1 when true a_j is not available
```

The chain runs: pullback degeneracy → acquisition gradient is 2D → perturbation along eigenvectors maximises starting-point diversity → (empirically) better L-BFGS optima → lower regret.

H001 tests the last two steps of this chain. H002 tests the middle part (natural gradient finding better acquisition optima). H003 tests the trust region design. The full chain is tested in H005.
