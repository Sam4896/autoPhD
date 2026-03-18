# Step 2: Build brain/invoke.py and brain/git_utils.py

**Prerequisites:** Step 1 (brain/budget.py must pass all tests)
**Output:** `brain/invoke.py`, `brain/git_utils.py`, `brain/security.py` (basic scanner)
**Test:** `python -m pytest tests/test_invoke_routing.py -v`

---

## What to build

`invoke.py` is the runtime dispatcher. It is called by GitHub Actions (`brain_listen.yml`) on every health/completion event. It:

1. Determines which agent(s) to invoke based on the event
2. Checks budget via `budget.py`
3. Assembles the input (specific files, capped to agent's `max_input_tokens`)
4. Reads the agent's frontmatter from `team/{agent}.md`
5. Calls the agent via `claude --print` (Claude) or Gemini Python SDK (Gemini agents)
6. Writes the output to the correct file
7. Logs the call to `agent_comms.jsonl` and appends to `agent_dialogue.md`
8. Commits and pushes the updated files to the experiment branch

---

## File to create: `brain/invoke.py`

### Public interface

```python
def invoke(
    event_type: str,           # From commit message: "health: RED", "complete:", etc.
    experiment_repo_path: Path, # Local checkout of the experiment repo
    brain_repo_path: Path,      # Local checkout of the autoresearch repo
    config: dict,              # Loaded from .autoresearch/config.json
    dry_run: bool = False,     # If True, don't commit/push (for testing)
) -> InvokeResult:
    """
    Main entry point. Called by GitHub Actions.
    Returns InvokeResult with status and summary.
    """
    ...
```

### Event routing table

```python
EVENT_ROUTING = {
    # Commit message pattern         → Agent(s) to invoke in order
    "health: RED":                   ["monitor", "bug_fixer"],   # monitor escalates to main if needed
    "health: YELLOW":                ["monitor"],                 # monitor may escalate to main
    "health: GREEN":                 [],                          # do nothing
    "complete: status=success":      ["analyst", "summarizer", "main"],
    "complete: status=interrupted":  ["analyst", "summarizer", "main"],
    "complete: status=SUCCESS_EARLY_STOP": ["analyst", "summarizer", "main"],
    "experiment: started":           ["main"],                    # initial setup + theory pre-check
    "brain: experiment_completed":   ["main"],                    # experiment agent finished
    "brain: bug_completed":          ["main"],                    # bug fixer finished
}
```

The routing is based on commit message prefix matching. If `kill_permanent: true` in `interrupt.json`, route to `[]` (do nothing).

**Completion signals** (written by Haiku code agents, read by invoke.py):
- `.autoresearch/experiment_completed.json` — Experiment Agent writes this after impl + smoke test
- `.autoresearch/bug_completed.json` — Bug Fixer Agent writes this after fix + smoke test
- `brain/invoke.py` reads these, commits them, and fires the appropriate routing event above

### Agent invocation

For Claude agents:
```bash
claude --print \
  --model {model_from_frontmatter} \
  --system "{agent_markdown_content}" \
  --message "{assembled_input_content}" \
  > {output_file}
```

For Gemini agents (Summarizer only):
```python
import google.generativeai as genai
model = genai.GenerativeModel("gemini-3.1-flash-lite-preview")
response = model.generate_content(system_prompt + "\n\n" + user_content)
```

### Input assembly

For each agent, assemble only the files listed in its frontmatter `reads` field, in order. Cap to `max_input_tokens`:

```python
def assemble_input(
    agent_name: str,
    agent_frontmatter: dict,
    experiment_repo_path: Path,
    brain_repo_path: Path,
) -> str:
    """
    Reads each file in frontmatter reads[], concatenates with headers.
    Enforces max_input_tokens by truncating files that exceed the cap.
    Always includes the most recent files at full size; truncates older/secondary files.
    """
    ...
```

### Logging to agent_comms.jsonl

After every agent call, append one JSON line:

```json
{
  "ts": "2026-03-17T14:23:01Z",
  "run_id": "{run_id}",
  "agent": "{agent_name}",
  "model": "{model}",
  "input_tokens": 1240,
  "output_tokens": 380,
  "input_files": [".autoresearch/heartbeat.json"],
  "output_file": ".autoresearch/monitor_report.md",
  "duration_ms": 4200,
  "cost_usd": 0.0041,
  "event": "YELLOW_HEARTBEAT",
  "action_taken": "ESCALATE_TO_MAIN",
  "trial": 1
}
```

### Committing output

After agent writes output:
```python
def commit_agent_outputs(
    repo_path: Path,
    files_changed: list[str],
    commit_message: str,   # "brain: {agent} {action} {run_id}"
) -> None:
    """Commits and pushes changed files to the experiment branch."""
    ...
```

---

## File to create: `brain/git_utils.py`

```python
"""
Git operations for AutoPhD brain/invoke.py.
Uses gitpython library. All functions raise GitError on failure.
"""

def git_commit_and_push(
    repo_path: Path,
    files: list[str],
    message: str,
    author_name: str = "AutoPhD Brain",
    author_email: str = "autophd@noreply.github.com",
) -> str:
    """Stages files, commits, pushes. Returns commit SHA."""
    ...

def git_pull(repo_path: Path) -> bool:
    """Pulls latest from remote. Returns True if new commits found."""
    ...

def git_read_file(
    repo_path: Path,
    file_path: str,
    max_lines: int = 0,     # 0 = all lines
    from_end: bool = False, # True = read last max_lines (tail)
) -> str:
    """Reads a file from the local checkout. Returns content as string."""
    ...

def git_create_branch(repo_path: Path, branch_name: str) -> None:
    """Creates and checks out a new branch."""
    ...

def github_create_pr(
    repo_url: str,
    head_branch: str,
    base_branch: str,
    title: str,
    body: str,
    github_token: str,
) -> str:
    """Creates a GitHub PR via gh CLI. Returns PR URL."""
    ...
```

---

## File to create: `brain/security.py` (basic version)

Implement the injection scanning from `team/security_agent.md`. This is the Python implementation of the Security Agent's injection scan logic (Job 2). It runs inline within invoke.py when processing EXPLORE mode web results, not as a separate agent call.

```python
"""
Prompt injection scanner for web search results.
See team/security_agent.md for full specification.
"""
import re
from dataclasses import dataclass

OVERRIDE_PATTERNS: list[str] = [...]   # from team/security_agent.md
ROLE_HIJACK_PATTERNS: list[str] = [...]
SCOPE_EXPAND_PATTERNS: list[str] = [...]
HARVEST_PATTERNS: list[str] = [...]

@dataclass
class ScanResult:
    clean: bool
    patterns_found: int
    sanitised_content: str
    summary: str  # e.g., "[SECURITY: 3 patterns removed. Categories: override, scope_expansion]"

def scan_web_content(content: str) -> ScanResult:
    """
    Scan and sanitize web search results before passing to Main Agent.
    Returns ScanResult with sanitised content and summary.
    If more than 5 patterns found: mark clean=False and reject entire result.
    """
    ...

def scan_code_diff(diff: str, allowed_paths: list[str], protected_paths: list[str]) -> ScanResult:
    """
    Scan a code diff for unsafe patterns.
    This implements the code safety check from team/security_agent.md Job 1.
    """
    ...
```

---

## File to create: `tests/test_invoke_routing.py`

```python
# All tests mock: subprocess.run (claude CLI), google.generativeai, git operations

# 1. RED heartbeat routes to monitor then bug_analyst_or_main
def test_routing_red_health():
    ...

# 2. GREEN heartbeat routes to nothing
def test_routing_green_health():
    ...

# 3. Complete event routes to analyst + summarizer + main
def test_routing_complete():
    ...

# 4. kill_permanent in interrupt.json → nothing runs
def test_kill_permanent_blocks_all():
    ...

# 5. Budget exceeded → nothing runs, NOTIFICATION.md written
def test_budget_exceeded_blocks_invocation():
    ...

# 6. agent_comms.jsonl is appended after every agent call
def test_agent_comms_written():
    ...

# 7. Claude agent receives correct model from frontmatter
def test_claude_model_from_frontmatter():
    ...

# 8. Gemini (gemini-3.1-flash-lite-preview) is called for summarizer (not claude CLI)
def test_gemini_used_for_summarizer():
    ...

# 9. Input assembly respects max_input_tokens
def test_input_assembly_token_cap():
    ...

# 10. injection scan blocks adversarial web content in EXPLORE mode
def test_injection_scan_blocks_adversarial():
    ...
```

---

## Acceptance criteria

- [ ] All 10 tests pass
- [ ] `ruff check brain/invoke.py brain/git_utils.py brain/security.py` returns 0 errors
- [ ] `mypy brain/invoke.py brain/git_utils.py brain/security.py --strict` returns 0 errors
- [ ] invoke.py never raises an uncaught exception — all errors are caught, logged, and written to state.md
- [ ] Every agent call writes to agent_comms.jsonl (verified by test 6)
- [ ] kill_permanent=true in interrupt.json stops ALL agent invocations (verified by test 4)
- [ ] No real API calls in tests (all mocked)
