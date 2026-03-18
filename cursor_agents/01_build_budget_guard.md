# Step 1: Build brain/budget.py

**Prerequisites:** None (first step)
**Output:** `brain/budget.py`
**Test:** `python -m pytest tests/test_budget.py -v`

---

## What to build

A budget enforcement module that is called before **every** agent invocation. It reads `.autoPhD/budget.json`, checks all limits, and returns a clear go/no-go decision with a reason.

---

## File to create: `brain/budget.py`

```python
"""
Budget enforcement for AutoPhD agent invocations.
Must be called before every agent call. No exceptions.
"""
from __future__ import annotations
import json
from dataclasses import dataclass
from enum import Enum
from pathlib import Path
from typing import Optional
import datetime


class BudgetStatus(Enum):
    OK = "ok"
    BLOCKED_TRIALS = "blocked_trials"
    BLOCKED_COST = "blocked_cost"
    BLOCKED_AGENT_TOKENS = "blocked_agent_tokens"
    BLOCKED_TOTAL_CALLS = "blocked_total_calls"
    BUDGET_FILE_MISSING = "budget_file_missing"
    BUDGET_FILE_CORRUPT = "budget_file_corrupt"


@dataclass
class BudgetCheckResult:
    status: BudgetStatus
    allowed: bool
    reason: str
    remaining_trials: Optional[int] = None
    remaining_cost_usd: Optional[float] = None
    remaining_agent_tokens: Optional[int] = None


def check_budget(
    budget_json_path: Path,
    agent_name: str,
    estimated_input_tokens: int,
    estimated_output_tokens: int,
    is_relaunch: bool = False,
) -> BudgetCheckResult:
    """
    Check if an agent invocation is within budget.

    Args:
        budget_json_path: Path to .autoPhD/budget.json
        agent_name: One of: main, experiment, monitor, analyst, summarizer,
                    tester, security, pr_agent, bug_analyst
        estimated_input_tokens: Estimated input tokens for this call
        estimated_output_tokens: Estimated output tokens for this call
        is_relaunch: True if this is an experiment relaunch (increments trial counter check)

    Returns:
        BudgetCheckResult with allowed=True if the call can proceed
    """
    # --- Load budget file ---
    # Must return BudgetStatus.BUDGET_FILE_MISSING if file doesn't exist
    # Must return BudgetStatus.BUDGET_FILE_CORRUPT if file is not valid JSON
    # or missing required fields

    # --- Check trial limit (only for experiment relaunches) ---
    # If is_relaunch=True and used.trials >= limits.n_experiment_trials:
    #   return BudgetCheckResult(status=BLOCKED_TRIALS, allowed=False, ...)

    # --- Check total call limit ---
    # If used.agent_calls >= limits.max_agent_calls_total:
    #   return BudgetCheckResult(status=BLOCKED_TOTAL_CALLS, allowed=False, ...)

    # --- Check per-agent token limit ---
    # estimated_total = estimated_input_tokens + estimated_output_tokens
    # if used.tokens_by_agent[agent_name] + estimated_total > limits.max_tokens_by_agent[agent_name]:
    #   return BudgetCheckResult(status=BLOCKED_AGENT_TOKENS, allowed=False, ...)

    # --- Check cost limit ---
    # estimated_cost = estimate_cost(agent_name, estimated_input_tokens, estimated_output_tokens)
    # if used.cost_usd + estimated_cost > limits.max_cost_usd:
    #   return BudgetCheckResult(status=BLOCKED_COST, allowed=False, ...)

    # --- All checks passed ---
    # return BudgetCheckResult(status=OK, allowed=True, ...)
    ...


def record_usage(
    budget_json_path: Path,
    agent_name: str,
    actual_input_tokens: int,
    actual_output_tokens: int,
    is_relaunch: bool = False,
) -> None:
    """
    Update budget.json with actual usage after a completed agent call.
    Must be called after every successful agent invocation.
    Must be atomic (write to temp file then rename, to avoid corruption on crash).
    """
    ...


def estimate_cost(
    agent_name: str,
    input_tokens: int,
    output_tokens: int,
) -> float:
    """
    Estimate cost in USD for a given agent call.
    Use these per-million-token rates:
    - claude-opus-4-6: input=$15, output=$75
    - claude-sonnet-4-6: input=$3, output=$15
    - gemini-2.0-flash-exp: input=$0, output=$0 (free tier)
    """
    ...


def get_budget_summary(budget_json_path: Path) -> str:
    """
    Return a human-readable one-line summary of current budget usage.
    Example: "Trials: 2/5 | Cost: $3.42/$10.00 | Calls: 18/50"
    """
    ...


def write_budget_exhausted_notification(
    budget_json_path: Path,
    state_md_path: Path,
    reason: str,
) -> None:
    """
    Write BUDGET_EXHAUSTED entry to state.md and NOTIFICATION.md.
    Called by invoke.py when check_budget returns blocked.
    """
    ...
```

---

## budget.json schema

The budget.json file must conform to `templates/config_template.json` `budget` section. Key fields:

```python
REQUIRED_BUDGET_FIELDS = [
    "run_id",
    "limits.n_experiment_trials",
    "limits.max_agent_calls_total",
    "limits.max_cost_usd",
    "limits.max_tokens_by_agent",
    "used.trials",
    "used.agent_calls",
    "used.cost_usd",
    "used.tokens_by_agent",
    "last_updated",
]

VALID_AGENT_NAMES = [
    "main", "experiment", "monitor", "analyst", "summarizer",
    "tester", "security", "pr_agent", "bug_analyst"
]
```

---

## File to create: `tests/test_budget.py`

Write tests for:

```python
# 1. Happy path — all checks pass
def test_check_budget_ok():
    # Create a budget.json with limits not reached
    # Call check_budget with a small estimated usage
    # Assert: allowed=True, status=OK

# 2. Trial limit exceeded
def test_check_budget_trial_limit():
    # Create budget.json with trials = n_experiment_trials
    # Call check_budget with is_relaunch=True
    # Assert: allowed=False, status=BLOCKED_TRIALS

# 3. Cost limit exceeded
def test_check_budget_cost_limit():
    # Create budget.json with cost_usd close to max_cost_usd
    # Call check_budget with estimated_output_tokens that would push over
    # Assert: allowed=False, status=BLOCKED_COST

# 4. Per-agent token limit exceeded
def test_check_budget_agent_token_limit():
    # Create budget.json with main tokens at limit
    # Call check_budget for main agent
    # Assert: allowed=False, status=BLOCKED_AGENT_TOKENS

# 5. Missing budget file
def test_check_budget_missing_file():
    # Call with a path that doesn't exist
    # Assert: allowed=False, status=BUDGET_FILE_MISSING

# 6. Corrupt budget file
def test_check_budget_corrupt_file():
    # Create a budget.json with missing required fields
    # Assert: allowed=False, status=BUDGET_FILE_CORRUPT

# 7. record_usage updates correctly
def test_record_usage():
    # Create budget.json
    # Call record_usage with known values
    # Re-read budget.json
    # Assert: used.tokens_by_agent["main"] increased by correct amount
    # Assert: used.cost_usd increased by estimated cost

# 8. record_usage is atomic (simulate partial write)
def test_record_usage_atomic():
    # Verify the write pattern uses temp-file-then-rename
    # (inspect the implementation or mock os.replace to confirm it's called)

# 9. estimate_cost returns correct values
def test_estimate_cost():
    # claude-opus-4-6: 1000 input, 1000 output → $0.015 + $0.075 = $0.090
    # claude-sonnet-4-6: 1000 input, 1000 output → $0.003 + $0.015 = $0.018
    # gemini: 1000 input, 1000 output → $0.00

# 10. Non-relaunch does not check trial limit
def test_check_budget_non_relaunch_ignores_trial_limit():
    # Create budget.json with trials at max
    # Call check_budget with is_relaunch=False
    # Assert: allowed=True (trial limit is not checked for non-relaunches)
```

---

## Acceptance criteria

- [ ] All 10 tests pass
- [ ] `ruff check brain/budget.py` returns 0 errors
- [ ] `mypy brain/budget.py --strict` returns 0 errors
- [ ] `record_usage` is atomic (uses temp file + rename pattern)
- [ ] `check_budget` never raises an exception — always returns a BudgetCheckResult
- [ ] No external API calls in budget.py (no anthropic, no google, no requests)
