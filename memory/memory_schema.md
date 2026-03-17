# Memory Schema — Mem0 Integration

This document specifies what gets stored in Mem0, when, by whom, and how to query it effectively.

---

## Design philosophy

Mem0 is long-term cross-experiment memory. Its job is to answer the question: "Have we seen something like this before?"

It is NOT a log. It is NOT a backup. It stores **reasoning and patterns**, not raw data.

Every entry should be immediately useful to the Main Agent when retrieved, without requiring the agent to read any other file.

---

## Memory entry types

### `experiment_result`

Written by Summarizer Agent after history condensation. One per completed experiment.

```json
{
  "agent": "summarizer_agent",
  "project": "riemannian-bo",
  "hypothesis": "H001",
  "run_id": "riemannian-bo-001",
  "type": "experiment_result",
  "content": "SUPPORTED: SA-RAASP achieved 18.3% ± 4.1% lower regret than uniform RAASP on Hartmann-6/100D rotated (p=0.003, Wilcoxon, n=20). Effect driven by 3.2× higher spread metric, correlating with ρ < 0.8 at convergence. Effect absent on axis-aligned variant (3.1% improvement, p=0.41).",
  "tags": ["hartmann6", "100d", "rotated", "sa_raasp", "turbo", "supported", "n_seeds=20"]
}
```

### `decision`

Written by Main Agent whenever it makes a non-trivial decision. Captures the reasoning, not just the outcome.

```json
{
  "agent": "main_agent",
  "project": "riemannian-bo",
  "hypothesis": "H001",
  "run_id": "riemannian-bo-001",
  "type": "decision",
  "content": "Interrupted run at iteration 87 due to ρ → 0.96 (rank-1 collapse). G_x had λ₂ < 0.002, indicating ∇μ ∥ ∇σ. Added Tikhonov regularisation ε = 1e-5 to Gram matrix. Relaunched. Hypothesis: this collapse is caused by trust region shrinking near convergence, not a code bug.",
  "tags": ["rank1_collapse", "tikhonov_regularisation", "interrupt", "rho_collapse"]
}
```

### `theory_check`

Written by Main Agent after theory pre-check or post-check.

```json
{
  "agent": "main_agent",
  "project": "riemannian-bo",
  "hypothesis": "H001",
  "run_id": null,
  "type": "theory_check",
  "content": "Pre-check: SA-RAASP spread dominance theorem (Theorem 5.1) is first-order only. The O(||δ||²/σ) correction from Christoffel symbols could in principle reduce spread on highly curved regions of the acquisition surface. However, G_x is nearly flat near x_best by construction (trust region is small), so the correction is negligible. Theorem applies.",
  "tags": ["theorem_5_1", "sa_raasp", "first_order", "christoffel_correction"]
}
```

### `pattern`

Written by Main Agent when it observes a cross-experiment pattern. May not be tied to a specific run.

```json
{
  "agent": "main_agent",
  "project": "riemannian-bo",
  "hypothesis": null,
  "run_id": null,
  "type": "pattern",
  "content": "ρ collapses toward 1 in the last 20% of the budget on all TuRBO runs so far (H001, H002). This appears to coincide with trust region contraction. When the TR side length drops below 0.01, ∇μ and ∇σ point in nearly the same direction. SA-RAASP and uniform RAASP both suffer this. Tikhonov regularisation in NGAO (ε = 1e-5) delays but does not prevent it.",
  "tags": ["rho_collapse", "trust_region", "turbo", "pattern", "convergence"]
}
```

### `warning`

Written by Main Agent when a config or hypothesis has a known risk.

```json
{
  "agent": "main_agent",
  "project": "riemannian-bo",
  "hypothesis": "H001",
  "run_id": "riemannian-bo-001",
  "type": "warning",
  "content": "Config riemannian-bo-001 used rotation seed=42. Three other teams have reported that random rotations with low prime seeds (seed < 100) on Hartmann-6 sometimes produce near-axis-aligned subspaces that favour axis-aligned methods. Recommend using seed > 1000 or verifying the rotation matrix has no near-zero off-diagonal blocks.",
  "tags": ["rotation_seed", "hartmann6", "axis_alignment_risk", "config_warning"]
}
```

---

## When each agent stores memories

| Agent | Stores when | What it stores |
|-------|-------------|----------------|
| Main Agent | After every significant reasoning step | `decision`, `theory_check`, `pattern`, `warning` |
| Summarizer Agent | After history condensation | `experiment_result` |
| Monitor Agent | Never | (Monitor writes reports, not memories) |
| Analyst Agent | Never | (Analyst writes reports, not memories) |
| Experiment Agent | Never | (Code decisions are in code comments, not Mem0) |

---

## Query strategy

At the start of every Main Agent session, query with multiple terms:

```python
# Pseudocode
queries = [
    f"hypothesis:{hypothesis_id}",           # This specific hypothesis
    f"benchmark:{benchmark_name}",            # This benchmark
    f"method:{method_name}",                  # This method
    f"type:warning project:{project}",        # Any warnings for this project
]

# For each query: retrieve top 3 results
# Deduplicate by run_id + type
# Keep top 10 by relevance score
# Load into context before reasoning
```

---

## Content quality standards

Every Mem0 entry must be:

**Self-contained** — a future agent reading only this entry should understand what happened. Do not write "see the state file for details" — include the critical numbers.

**Specific** — include actual numbers, not vague descriptions. Not "the method improved significantly" but "18.3% ± 4.1%, p=0.003".

**Actionable** — if a future agent reads this, what should it do differently? Make that explicit.

**Concise** — max 100 words in `content`. Mem0 entries appear alongside many others; brevity is essential.

---

## mem0_config_template.json

See `memory/mem0_config_template.json` for API configuration. Copy to `memory/mem0_config.json` and fill in your API key (do not commit this file).

---

## memory_bridge.py interface

```python
from memory.memory_bridge import MemoryBridge

bridge = MemoryBridge(config_path="memory/mem0_config.json")

# Store a memory
bridge.store({
    "agent": "main_agent",
    "project": "riemannian-bo",
    "hypothesis": "H001",
    "run_id": "riemannian-bo-001",
    "type": "decision",
    "content": "...",
    "tags": ["tag1", "tag2"]
})

# Query memories
results = bridge.query(
    queries=["hypothesis:H001", "benchmark:hartmann6"],
    top_k=5
)
for r in results:
    print(r["content"])
```

The bridge handles deduplication, relevance ranking, and error handling. Never call the Mem0 API directly from agent code.
