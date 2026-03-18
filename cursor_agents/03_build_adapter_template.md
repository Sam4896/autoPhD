# Step 3: Build templates/autoPhD_adapter.py.template

**Prerequisites:** None (independent of steps 1–2 — can be built in parallel)
**Output:** `templates/autoPhD_adapter.py.template`
**Test:** `python -m pytest tests/test_adapter_template.py -v`

---

## What to build

`autoPhD_adapter.py` is the **only file added to the experiment repo's root**. It wraps `run_experiment.py` and handles all git communication. It must be:

- Self-contained (no imports from the brain repo)
- Minimal (under 200 lines of real code)
- Robust (handles network failures gracefully — git push can fail)
- Parameterised (template variables filled in by `brain/connect.py`)

---

## Template variables (filled by brain/connect.py)

```python
# These are replaced in the template when generating the actual file
RUN_ID = "{{RUN_ID}}"                          # e.g., "exp-riemannian-bo-H001-20260317-143022"
EXPERIMENT_BRANCH = "{{EXPERIMENT_BRANCH}}"    # e.g., "exp/riemannian-bo-H001-20260317-143022"
AUTORESEARCH_DIR = "{{AUTORESEARCH_DIR}}"      # path to .autoPhD/ in the repo
HEALTH_CHECK_INTERVAL = {{HEALTH_CHECK_INTERVAL}}   # seconds, default 30
GIT_PULL_INTERVAL = {{GIT_PULL_INTERVAL}}           # seconds, default 30
```

---

## File to create: `templates/autoPhD_adapter.py.template`

The template must implement exactly this behavior:

### 1. Startup commit

When the adapter starts, before launching run_experiment.py:
```python
# Write initial state to .autoPhD/state.md (append "experiment: started" line)
# git add .autoPhD/
# git commit -m "experiment: started {RUN_ID} on {EXPERIMENT_BRANCH}"
# git push origin {EXPERIMENT_BRANCH}
```

This commit triggers brain_listen.yml for the first time.

### 2. Launch experiment subprocess

```python
proc = subprocess.Popen(
    [sys.executable, run_experiment_file] + extra_args,
    stdout=sys.stdout,  # pass through to terminal
    stderr=sys.stderr,
)
```

### 3. Background health thread

```python
def health_loop():
    """
    Runs in background thread (daemon=True).
    Every HEALTH_CHECK_INTERVAL seconds:
      1. Read log.csv if it exists
      2. Run health_checker.py checks
      3. If health changed: write heartbeat.json, git commit + push
      4. Every GIT_PULL_INTERVAL seconds: git pull + check for interrupt.json
      5. If interrupt.json found: write /tmp/{RUN_ID}.interrupt
    """
    last_health = "GREEN"
    last_pull_time = time.time()

    while True:
        time.sleep(HEALTH_CHECK_INTERVAL)

        # Read logs and compute health
        health, flags = compute_health()

        # Commit heartbeat if health changed or on every 5th check
        if health != last_health or check_count % 5 == 0:
            write_heartbeat(health, flags)
            git_commit_heartbeat(health, flags)

        # Pull for interrupt on interval
        if time.time() - last_pull_time > GIT_PULL_INTERVAL:
            git_pull_for_interrupt()
            last_pull_time = time.time()

        last_health = health
```

### 4. Interrupt polling in main thread

```python
# Check for /tmp/{RUN_ID}.interrupt written by health thread
INTERRUPT_SIGNAL = Path(f"/tmp/{RUN_ID}.interrupt")

# While experiment runs, check for interrupt signal
while proc.poll() is None:
    if INTERRUPT_SIGNAL.exists():
        proc.terminate()
        proc.wait(timeout=10)
        write_final_heartbeat("interrupted")
        git_commit_completion("interrupted")
        sys.exit(0)
    time.sleep(1)
```

### 5. Completion handling

```python
exit_code = proc.wait()
if exit_code == 0:
    write_completion_flag("success")
    git_commit_completion("success")
elif exit_code != 0:
    write_completion_flag("error")
    git_commit_completion("error")
```

### 6. Git operations (inline, no gitpython dependency)

Use only `subprocess.run(["git", ...])` — no external git libraries. The adapter must work with just Python stdlib + git binary.

```python
def git_push_with_retry(max_retries: int = 3, backoff: float = 5.0) -> bool:
    """Push with retry on network failure. Returns True on success."""
    for attempt in range(max_retries):
        result = subprocess.run(["git", "push", "origin", EXPERIMENT_BRANCH], ...)
        if result.returncode == 0:
            return True
        time.sleep(backoff * (2 ** attempt))
    return False  # Push failed — log but continue (don't crash the experiment)
```

**Critical**: Git push failures must NEVER crash the experiment. Log the failure and continue.

### 7. Usage

```bash
# Instead of: python run_experiment.py config.json
# Use:        python autoPhD_adapter.py run_experiment.py config.json
```

All arguments after `autoPhD_adapter.py` are passed directly to `run_experiment.py`.

---

## health_checker.py interface

The adapter calls `health_checker.py` as a subprocess to compute health. The interface is:

```bash
python health_checker.py \
  --log .autoPhD/log.csv \
  --diagnostics .autoPhD/diagnostics.csv \
  --config .autoPhD/config.json \
  --output .autoPhD/heartbeat.json
# Exits with code 0. Writes heartbeat.json. Stdout is suppressed.
```

The adapter reads `heartbeat.json` to get the `health` field.

---

## File to create: `tests/test_adapter_template.py`

Test the template by instantiating it with test parameters and running it:

```python
# Setup: generate adapter from template with test values
# Use a minimal fake run_experiment.py that writes 5 log rows then exits

# 1. Adapter writes startup commit message
def test_startup_commit():
    # Run adapter with fake experiment
    # Check git log for "experiment: started" commit
    ...

# 2. Adapter passes arguments through to run_experiment.py
def test_argument_passthrough():
    # Fake run_experiment.py writes sys.argv to a file
    # Adapter should pass --config arg through
    ...

# 3. Health thread commits heartbeat on health change
def test_health_commit_on_change():
    # Fake health_checker.py returns GREEN then YELLOW
    # Check git log for health commit
    ...

# 4. Interrupt signal stops experiment cleanly
def test_interrupt_stops_cleanly():
    # Write /tmp/{run_id}.interrupt after 1 second
    # Verify experiment subprocess is terminated
    # Verify completion commit with status=interrupted
    ...

# 5. Git push failure does not crash experiment
def test_push_failure_is_non_fatal():
    # Mock git push to return non-zero
    # Verify experiment still completes normally
    ...

# 6. Normal completion writes completion commit
def test_normal_completion_commit():
    # Run adapter to completion
    # Check git log for "complete: status=success" commit
    ...
```

---

## Acceptance criteria

- [ ] All 6 tests pass
- [ ] Adapter is < 200 lines of code (not counting comments and blank lines)
- [ ] Adapter has NO imports from brain/ (fully self-contained)
- [ ] Git push failure does not crash the experiment (test 5)
- [ ] Template variables are clearly marked with `{{...}}` syntax
- [ ] Works on Linux/macOS (cluster environment) — no Windows-only paths
- [ ] health_checker.py is called as subprocess (not imported) — this is key for isolation
