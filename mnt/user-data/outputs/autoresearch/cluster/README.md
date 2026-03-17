# Cluster Scripts

This directory contains scripts that run on the cluster. They have no LLM access. They are fast, stateless, and defensive.

---

## health_check.py

The heartbeat generator. Runs every 30 seconds (via cron or a background process). Reads `log.csv` and `diagnostics.csv`, evaluates default and custom checks, writes `heartbeat.json`.

### Design constraints

- **No LLM calls** — must complete in < 2 seconds on a loaded cluster node
- **No network access** — reads and writes local files only (git push is handled by watcher.py)
- **Graceful on missing data** — if the log doesn't exist yet, writes `health: green` with `iteration: 0`
- **Idempotent** — running twice in a row produces the same result
- **Never crashes** — wraps all I/O in try/except; any unexpected error produces `health: yellow` with the error in flags

### Responsibilities

1. Read `config.json` to get: `run_id`, `budget`, `n_seeds`, `timeout_minutes`, `diagnostic_freq`, paths
2. Read `log.csv` (last 50 rows for efficiency)
3. Read `diagnostics.csv` (last 30 rows)
4. Load `custom_checks.py` from the experiment's src directory (if it exists)
5. Evaluate default checks
6. Evaluate custom checks
7. Determine overall health: any RED → RED; any YELLOW and no RED → YELLOW; else GREEN
8. Write `heartbeat.json`

### Default check implementations

```python
# Pseudocode — see actual implementation in health_check.py

def check_nan_in_regret(log_df):
    if log_df["regret"].isna().any():
        return "red", "nan_in_regret: NaN found in regret column"
    return "green", None

def check_seed_crash(log_df, config):
    expected_seeds = set(config["seeds"])
    observed_seeds = set(log_df["seed"].unique())
    crashed = expected_seeds - observed_seeds
    if crashed and len(log_df) > config["n_seeds"] * 5:
        # Only flag after at least 5 iterations per seed
        return "red", f"seed_crash: seeds {sorted(crashed)} not in log"
    return "green", None

def check_timeout_imminent(elapsed, timeout):
    if elapsed > 0.9 * timeout:
        return "red", f"timeout_imminent: {elapsed:.1f}/{timeout} min ({elapsed/timeout*100:.0f}%)"
    if elapsed > 0.8 * timeout:
        return "yellow", f"timeout_warning: {elapsed:.1f}/{timeout} min"
    return "green", None

def check_data_stale(last_timestamp):
    age_minutes = (datetime.now() - parse(last_timestamp)).total_seconds() / 60
    if age_minutes > 5:
        return "red", f"data_stale: last write {age_minutes:.1f} min ago"
    return "green", None

def compute_regret_slope(log_df, window=30):
    recent = log_df.tail(window)
    if len(recent) < 5:
        return None
    x = recent["iteration"].values
    y = recent["regret"].values
    # Simple linear regression
    slope = (len(x) * (x * y).sum() - x.sum() * y.sum()) / \
            (len(x) * (x**2).sum() - x.sum()**2 + 1e-12)
    return slope
```

### Custom check loading

```python
import importlib.util

def load_custom_checks(src_dir: str):
    checks_path = Path(src_dir) / "custom_checks.py"
    if not checks_path.exists():
        return []
    
    spec = importlib.util.spec_from_file_location("custom_checks", checks_path)
    module = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(module)
    
    if not hasattr(module, "get_custom_checks"):
        return []
    
    return module.get_custom_checks()

def run_custom_checks(checks, log_df, diagnostics_df, config):
    results = []
    for check in checks:
        try:
            triggered, message = check["fn"](log_df, config)
            if triggered:
                severity = check["severity"]
                results.append((severity, f"{check['name']}: {message}"))
        except Exception as e:
            results.append(("yellow", f"{check['name']}: ERROR — {e}"))
    return results
```

---

## watcher.py

Monitors `heartbeat.json` and `completion_flag.json` for changes. Triggers automated responses.

### Responsibilities

1. **Git sync** — push new log data to the remote repo every N seconds (default: 60)
2. **Heartbeat response** — when health turns RED, set `interrupt.json`
3. **Completion response** — when `completion_flag.json` appears, write a completion notification to `watcher_log.txt`
4. **No LLM invocation** — the watcher never calls an LLM; it only manipulates files

### Watcher behaviour table

| Event | Action |
|-------|--------|
| health: green | Git push (if changes); nothing else |
| health: yellow | Git push; write to `watcher_log.txt` |
| health: red (n_consecutive >= threshold) | Git push; set `interrupt.json`; write to `watcher_log.txt` |
| `completion_flag.json` appears | Git push; write to `watcher_log.txt`; exit watcher loop |
| Git push fails | Write error to `watcher_log.txt`; retry next cycle |

### watcher_log.txt format

```
[2026-03-17T14:23:01Z] HEALTH_CHANGE green → yellow: flags=[warn_plateau]
[2026-03-17T14:25:01Z] HEALTH_CHANGE yellow → red: flags=[nan_in_regret]
[2026-03-17T14:25:02Z] INTERRUPT_SET: interrupt.json written (reason: nan_in_regret)
[2026-03-17T14:25:30Z] GIT_PUSH: success (commit abc1234)
[2026-03-17T14:45:00Z] COMPLETION: completion_flag.json detected — experiment complete
```

This log is the audit trail for automated actions. It is not processed by any LLM.

---

## launcher.py

Validates `config.json` and launches `run_experiment.py`.

### Responsibilities

1. Poll for `config.json` with `"status": "ready_to_launch"`
2. Validate the config (required fields, path existence, budget bounds)
3. Launch `run_experiment.py` in a subprocess with the config path
4. Monitor the subprocess; if it exits unexpectedly, write to `watcher_log.txt`
5. Set `config.json: {"status": "running"}` on successful launch

### Validation checks

```python
REQUIRED_FIELDS = [
    "run_id", "status", "hypothesis", "method_name", "baseline_name",
    "benchmark_name", "benchmark_dim", "budget", "n_seeds",
    "seeds", "kernel", "acquisition", "timeout_minutes", "paths"
]

def validate_config(config):
    errors = []
    for field in REQUIRED_FIELDS:
        if field not in config:
            errors.append(f"Missing required field: {field}")
    
    if config.get("budget", 0) > 10000:
        errors.append(f"Budget {config['budget']} exceeds safety limit 10000")
    
    if config.get("n_seeds", 0) > 100:
        errors.append(f"n_seeds {config['n_seeds']} exceeds safety limit 100")
    
    src_dir = config.get("paths", {}).get("src_dir", "")
    run_script = Path(src_dir) / "run_experiment.py"
    if not run_script.exists():
        errors.append(f"run_experiment.py not found at {run_script}")
    
    return errors
```

If validation fails, set `config.json: {"status": "failed", "error": "<error list>"}` and do not launch.

---

## Cron setup

On the cluster, add to crontab:

```cron
# health_check.py every 30 seconds (run twice per minute)
* * * * * cd /path/to/autoresearch && python cluster/health_check.py path/to/config.json
* * * * * sleep 30 && cd /path/to/autoresearch && python cluster/health_check.py path/to/config.json

# watcher.py as a persistent background process
# Start manually: nohup python cluster/watcher.py path/to/config.json &
```

Or run `watcher.py` in a `screen` or `tmux` session for easier monitoring.
