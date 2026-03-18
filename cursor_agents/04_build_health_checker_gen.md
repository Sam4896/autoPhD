# Step 4: Build health_checker.py.j2 and brain/health_checker_gen.py

**Prerequisites:** Step 1 (budget.py), Step 3 (adapter template)
**Output:** `templates/health_checker.py.j2`, `brain/health_checker_gen.py`
**Test:** `python -m pytest tests/test_health_checker_gen.py -v`

---

## What to build

`health_checker_gen.py` reads `projects/{project}/health_thresholds.md` (the user-filled file) and generates a working `health_checker.py` using the Jinja2 template `health_checker.py.j2`.

The generated `health_checker.py` is committed to the experiment repo during `autoresearch connect`. It runs on the cluster as a subprocess called by `autoresearch_adapter.py`.

---

## health_checker.py interface (what gets generated)

The generated file must accept this command line:

```bash
python health_checker.py \
  --log path/to/log.csv \
  --diagnostics path/to/diagnostics.csv \
  --config path/to/config.json \
  --output path/to/heartbeat.json
```

Writes `heartbeat.json` and exits with code 0.

### heartbeat.json format

```json
{
  "timestamp": "2026-03-17T14:23:01Z",
  "run_id": "exp-H001-20260317",
  "iteration": 87,
  "budget": 200,
  "pct_budget_used": 43.5,
  "health": "yellow",
  "flags": ["plateau_warning: slope=0.0001 (threshold -0.001)"],
  "success_criteria_met": false,
  "success_details": null,
  "primary_metric_current": 0.234,
  "primary_metric_trend": "plateau",
  "elapsed_minutes": 14.2,
  "timeout_minutes": 90,
  "seeds_active": 20,
  "seeds_expected": 20
}
```

---

## File to create: `templates/health_checker.py.j2`

Jinja2 template that generates the health_checker.py. Template variables come from the parsed health_thresholds.md.

Key sections the template must include:

### Section 1: Imports and CLI parsing (static, no Jinja)
```python
#!/usr/bin/env python3
"""Generated health checker — do not edit manually. Regenerate from health_thresholds.md."""
import argparse, json, sys
from datetime import datetime
from pathlib import Path
import pandas as pd
import numpy as np
```

### Section 2: Check functions (generated from thresholds)

For each `green_criteria` item in health_thresholds.md:
```jinja
{% for check in green_criteria if check.enabled %}
def check_green_{{ check.name }}(log_df, config):
    """{{ check.description }}"""
    # Generated from: {{ check.condition }}
    ...
    return triggered, message
{% endfor %}
```

For each `yellow_criteria` item:
```jinja
{% for check in yellow_criteria if check.enabled %}
def check_yellow_{{ check.name }}(log_df, diagnostics_df, config):
    """{{ check.description }}"""
    ...
    return triggered, message
{% endfor %}
```

For each `red_criteria` item:
```jinja
{% for check in red_criteria if check.enabled %}
def check_red_{{ check.name }}(log_df, diagnostics_df, config):
    """{{ check.description }}"""
    ...
    return triggered, message
{% endfor %}
```

### Section 3: Main evaluation logic (mostly static)
```python
def evaluate_health(log_df, diagnostics_df, config):
    """Run all checks and return (health_level, flags)."""
    # Run red checks first
    # Then yellow checks
    # Then green checks
    # Return highest severity
    ...
```

### Section 4: heartbeat.json writer (static)

---

## File to create: `brain/health_checker_gen.py`

```python
"""
Reads health_thresholds.md, parses threshold specifications,
and generates health_checker.py via health_checker.py.j2.
"""
from pathlib import Path
import yaml
import re
from jinja2 import Environment, FileSystemLoader
from dataclasses import dataclass
from typing import Optional

@dataclass
class ThresholdCheck:
    name: str
    description: str
    condition: str
    enabled: bool
    severity: str  # green, yellow, red
    # Parsed fields from condition:
    column_name: Optional[str]
    operator: Optional[str]
    threshold: Optional[float]
    window: Optional[int]

def parse_health_thresholds(thresholds_md_path: Path) -> dict:
    """
    Parse health_thresholds.md and return a dict suitable for Jinja2 rendering.
    Must handle:
    - Missing sections (treat as empty)
    - enabled: false items (skip)
    - null values in threshold fields
    Returns dict with keys: green_criteria, yellow_criteria, red_criteria, success_criteria, config
    """
    ...

def generate_health_checker(
    thresholds_md_path: Path,
    template_path: Path,
    output_path: Path,
    config: dict,  # from config_template.json — for run_id, budget, etc.
) -> None:
    """
    Generates health_checker.py from template and thresholds.
    Writes to output_path.
    """
    ...

def validate_generated_health_checker(generated_path: Path) -> tuple[bool, str]:
    """
    Validates that the generated health_checker.py:
    1. Is valid Python syntax (compile it)
    2. Can be imported without errors
    3. Has the required CLI interface (--log, --diagnostics, --config, --output args)
    Returns (valid: bool, error: str)
    """
    ...
```

---

## File to create: `tests/test_health_checker_gen.py`

```python
# Use the sample health_thresholds.md in tests/fixtures/

# 1. Parse minimal thresholds file (only required sections)
def test_parse_minimal_thresholds():
    # Parse a thresholds file with just nan_in_metric enabled
    # Assert: red_criteria has 1 item, others have 0 enabled items
    ...

# 2. Parse full thresholds file (all sections)
def test_parse_full_thresholds():
    # Parse tests/fixtures/full_health_thresholds.md
    # Assert all enabled checks are present
    ...

# 3. Parse ignores enabled: false items
def test_parse_ignores_disabled_checks():
    # thresholds.md with 3 yellow checks, 2 disabled
    # Assert: only 1 yellow check in output
    ...

# 4. Generate health_checker.py from minimal thresholds
def test_generate_minimal():
    # Call generate_health_checker with minimal thresholds
    # Assert: output file exists, is valid Python
    ...

# 5. Generated file has correct CLI interface
def test_generated_cli_interface():
    # Generate a health_checker.py
    # Run: python health_checker.py --help
    # Assert: --log, --diagnostics, --config, --output args present
    ...

# 6. Generated file runs on sample log.csv and writes heartbeat.json
def test_generated_runs():
    # Generate health_checker.py
    # Create minimal log.csv with 10 rows (no NaN, steady regret)
    # Run: python health_checker.py --log ... --output ...
    # Assert: heartbeat.json exists with health="green"
    ...

# 7. Generated file detects NaN → health=red
def test_generated_detects_nan():
    # log.csv with NaN in regret column
    # Run health_checker
    # Assert: heartbeat.json health="red", flags contains "nan_in_metric"
    ...

# 8. validate_generated_health_checker catches syntax errors
def test_validate_catches_syntax_error():
    # Write a Python file with a syntax error
    # Assert: validate returns (False, error_message)
    ...
```

Also create `tests/fixtures/full_health_thresholds.md` with a complete example using all sections.

---

## Acceptance criteria

- [ ] All 8 tests pass
- [ ] `ruff check brain/health_checker_gen.py` returns 0 errors
- [ ] Generated health_checker.py is valid Python (test 4)
- [ ] Generated health_checker.py runs on sample data (test 6)
- [ ] Generated health_checker.py detects NaN (test 7)
- [ ] Disabled checks are not present in generated code (test 3)
- [ ] Generated file has no imports from brain/ (fully self-contained)
