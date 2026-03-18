# Step 6: Build brain/connect.py — the `autoPhD connect` CLI

**Prerequisites:** Steps 1–5 (all previous steps must be complete and tested)
**Output:** `brain/connect.py`
**Test:** Integration test (see below — requires a real GitHub repo)

---

## What to build

`brain/connect.py` is the single command that links the brain to an experiment repo. It is the "installation" process. Run it once per experiment.

```bash
autoPhD connect \
  --repo https://github.com/you/exp-repo \
  --branch exp/riemannian-bo-H001-20260317-143022 \
  --entry run_experiment.py \
  --hypothesis projects/riemannian-bo/hypotheses/H001_SA_RAASP.md \
  --project riemannian-bo
```

---

## Full sequence of connect.py

```
1. Validate inputs
   - GitHub repo URL is accessible (via gh CLI)
   - Branch exists in the repo
   - run_experiment.py exists in the repo
   - Hypothesis file exists in the brain repo
   - health_thresholds.md exists in projects/{project}/

2. Parse hypothesis file
   - Extract AutoPhD integration settings (exploration_mode, allowed_paths, protected_paths, approval_mode, budget)
   - Validate all required fields present

3. Generate config.json
   - From templates/config_template.json
   - Fill in: run_id, hypothesis_id, project, branch, repo URLs, budget settings
   - Write to a temp location

4. Invoke Main Agent for initial setup
   - Call: claude --print --model claude-opus-4-6 with hypothesis + project.md
   - Main Agent produces: theory pre-check + initial state.md entries
   - Budget check first (for this initial call)

5. Generate health_checker.py
   - Call health_checker_gen.generate_health_checker()
   - Validate generated file

6. Instantiate adapter template
   - Fill in template variables: RUN_ID, EXPERIMENT_BRANCH, etc.
   - Write autoPhD_adapter.py

7. Instantiate GitHub Actions templates
   - Fill in {{BRAIN_REPO}} with the brain repo URL
   - Write .github/workflows/brain_listen.yml
   - Write .github/workflows/experiment_resume.yml

8. Create .autoPhD/ directory structure
   - Write config.json
   - Write initial state.md (from state_template.md + Main Agent output)
   - Write budget.json (initial — all zeros used)
   - Create empty agent_comms.jsonl
   - Create empty agent_dialogue.md

9. Set GitHub secrets
   - ANTHROPIC_API_KEY (read from local env)
   - GEMINI_API_KEY (read from local env)
   - If brain repo is private: AUTORESEARCH_READ_TOKEN (read from local env or ask user)
   - Use: gh secret set --repo {repo} --body {value}

10. Commit and push everything to the experiment branch
    - git add autoPhD_adapter.py health_checker.py .autoPhD/ .github/
    - git commit -m "autoPhD: connect H{NNN} brain to experiment repo"
    - git push origin {branch}

11. Print success message
    - What was created
    - How to run the experiment: python autoPhD_adapter.py run_experiment.py .autoPhD/config.json
    - Where to watch progress: GitHub Actions tab + .autoPhD/agent_dialogue.md
```

---

## CLI interface (use Click library)

```python
import click
from pathlib import Path

@click.command()
@click.option("--repo", required=True, help="GitHub URL of the experiment repo")
@click.option("--branch", required=True, help="Experiment branch name (exp/...)")
@click.option("--entry", required=True, help="Entry point script (e.g., run_experiment.py)")
@click.option("--hypothesis", required=True, help="Path to hypothesis file in brain repo")
@click.option("--project", required=True, help="Project name (e.g., riemannian-bo)")
@click.option("--dry-run", is_flag=True, default=False, help="Print what would be done without doing it")
@click.option("--autonomous", is_flag=True, default=False, help="Set approval_mode to autonomous")
def connect(repo, branch, entry, hypothesis, project, dry_run, autonomous):
    """Connect AutoPhD brain to an experiment repository."""
    ...
```

Also implement these commands:

```python
@click.command()
@click.option("--repo", required=True)
@click.option("--branch", required=True)
@click.option("--reason", default="user_kill")
def kill(repo, branch, reason):
    """Kill a running experiment permanently."""
    # Write interrupt.json with kill_permanent=True
    # Commit and push to branch
    ...

@click.command()
@click.option("--repo", required=True)
@click.option("--branch", required=True)
def status(repo, branch):
    """Show current experiment status."""
    # git fetch + read .autoPhD/state.md last 10 lines
    # Read .autoPhD/budget.json and print summary
    # Read .autoPhD/heartbeat.json last entry
    ...

# Group all commands
@click.group()
def cli():
    """AutoPhD — AI research team for your experiment repo."""
    pass

cli.add_command(connect)
cli.add_command(kill)
cli.add_command(status)
```

---

## Integration test procedure

(Cannot be fully unit-tested — requires real GitHub access)

```bash
# 1. Create a minimal test experiment repo
mkdir test-exp && cd test-exp
git init
cat > run_experiment.py << 'EOF'
import sys, json, csv, time
from pathlib import Path
config = json.loads(Path(sys.argv[1]).read_text())
log_path = Path(config["paths"]["log"])
with open(log_path, "w") as f:
    f.write("run_id,seed,iteration,regret,best_y,timestamp,status,method\n")
    for i in range(10):
        f.write(f"{config['run_id']},0,{i},{1.0/(i+1):.4f},{i+1},2026-01-01T00:00:00Z,normal,test_method\n")
        time.sleep(0.5)
EOF
git add run_experiment.py
git commit -m "initial"
git remote add origin https://github.com/{you}/test-exp.git
git push -u origin main
git checkout -b exp/test-H001-20260317-143022
git push -u origin exp/test-H001-20260317-143022

# 2. Set API keys
export ANTHROPIC_API_KEY="..."
export GEMINI_API_KEY="..."

# 3. Run connect
python brain/connect.py \
  --repo https://github.com/{you}/test-exp \
  --branch exp/test-H001-20260317-143022 \
  --entry run_experiment.py \
  --hypothesis projects/riemannian-bo/hypotheses/H001_SA_RAASP.md \
  --project riemannian-bo \
  --dry-run   # First do a dry run

# 4. If dry run looks good, run for real
python brain/connect.py \
  --repo https://github.com/{you}/test-exp \
  --branch exp/test-H001-20260317-143022 \
  --entry run_experiment.py \
  --hypothesis projects/riemannian-bo/hypotheses/H001_SA_RAASP.md \
  --project riemannian-bo

# 5. Verify created files
gh repo clone {you}/test-exp /tmp/verify-exp -- --branch exp/test-H001-20260317-143022
ls /tmp/verify-exp/autoPhD_adapter.py    # should exist
ls /tmp/verify-exp/health_checker.py          # should exist
ls /tmp/verify-exp/.autoPhD/config.json  # should exist
ls /tmp/verify-exp/.github/workflows/brain_listen.yml  # should exist

# 6. Verify secrets were set
gh secret list --repo {you}/test-exp
# Should show: ANTHROPIC_API_KEY, GEMINI_API_KEY
```

---

## Acceptance criteria

- [ ] `autoPhD connect --dry-run` prints all planned actions without writing anything
- [ ] `autoPhD connect` creates all required files (adapter, health_checker, .autoPhD/, .github/workflows/)
- [ ] `autoPhD kill` writes interrupt.json with kill_permanent=True and pushes
- [ ] `autoPhD status` shows current health and budget summary
- [ ] All secrets are set via gh CLI (verified by `gh secret list`)
- [ ] Generated `run_id` includes hypothesis ID and timestamp (no duplicates on re-run)
- [ ] `--dry-run` flag works and prints exactly what would be done
- [ ] Error messages are clear when: repo not found, branch not found, hypothesis file not found, missing API keys
