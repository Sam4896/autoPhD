# AutoPhD — Cursor Agent Build Guide

This directory contains implementation specs for building the AutoPhD brain system.
Each file is a self-contained instruction for a coding agent (Cursor, Claude, or similar).

**Read `ARCHITECTURE.md` and all `team/*.md` files before using any spec here.**

---

## How to use these specs

1. Open this repo in Cursor
2. Open one spec file (e.g., `01_build_budget_guard.md`)
3. Ask Cursor agent: "Implement this spec exactly as described"
4. Run the tests specified in the spec
5. Move to the next spec only when tests pass

**Do NOT skip steps or combine multiple specs into one session.** Each spec is independently testable.

---

## Build Order (strict — each step depends on the previous)

| Step | Spec file | Output | Test command |
|------|-----------|--------|--------------|
| 1 | `01_build_budget_guard.md` | `brain/budget.py` | `python -m pytest tests/test_budget.py` |
| 2 | `02_build_invoke_layer.md` | `brain/invoke.py`, `brain/git_utils.py` | `python -m pytest tests/test_invoke_routing.py` |
| 3 | `03_build_adapter_template.md` | `templates/autoresearch_adapter.py.template` | `python -m pytest tests/test_adapter_template.py` |
| 4 | `04_build_health_checker_gen.md` | `templates/health_checker.py.j2`, `brain/health_checker_gen.py` | `python -m pytest tests/test_health_checker_gen.py` |
| 5 | `05_build_github_actions.md` | `templates/brain_listen.yml`, `templates/experiment_resume.yml` | Manual test with test repo |
| 6 | `06_build_connect_cli.md` | `brain/connect.py` | Integration test (see spec) |
| 7 | `07_build_test_suite.md` | `tests/`, `ci/` | `python -m pytest tests/ -v` |

---

## Shared conventions

All Python code in `brain/` must:
- Use Python 3.11+
- Have type hints on all function signatures
- Pass `ruff check` (linting)
- Pass `mypy` (type checking, strict mode)
- Have docstrings on all public functions

All test files in `tests/` must:
- Use `pytest`
- Be named `test_{module}.py`
- Cover the happy path + 2 failure modes per function
- Not make real API calls (mock `anthropic`, `google.generativeai`, `subprocess`)
- Complete in < 5 seconds total

---

## Environment setup

```bash
# Install dependencies
pip install anthropic google-generativeai pyyaml jinja2 gitpython click pytest mypy ruff

# Set environment variables (for real usage — tests use mocks)
export ANTHROPIC_API_KEY="..."
export GEMINI_API_KEY="..."
export GITHUB_TOKEN="..."
```

---

## File structure being built

```
brain/
├── __init__.py
├── budget.py          ← step 1
├── invoke.py          ← step 2
├── git_utils.py       ← step 2
├── security.py        ← step 2 (basic version)
├── health_checker_gen.py  ← step 4
└── connect.py         ← step 6

templates/
├── autoresearch_adapter.py.template  ← step 3
├── health_checker.py.j2              ← step 4
├── brain_listen.yml                  ← step 5
└── experiment_resume.yml             ← step 5

tests/
├── test_budget.py              ← step 1 output
├── test_invoke_routing.py      ← step 2 output
├── test_adapter_template.py    ← step 3 output
├── test_health_checker_gen.py  ← step 4 output
└── (more tests in step 7)

ci/
├── validate_agent_frontmatter.py  ← step 7
├── validate_templates.py          ← step 7
└── validate_cursor_agents.py      ← step 7
```

---

## Verification commands (run after all steps complete)

```bash
# Full test suite
python -m pytest tests/ -v --tb=short

# Linting
ruff check brain/ tests/ ci/

# Type checking
mypy brain/ --strict

# Validate all agent markdown files
python ci/validate_agent_frontmatter.py

# Validate all templates
python ci/validate_templates.py
```

All commands must pass with 0 errors before the system is considered buildable.
