# AutoPhD — System Architecture v1.0

Complete reference for the AutoPhD Brain system. Read this before modifying any component.

---

## Table of Contents

1. [Philosophy and Core Principles](#1-philosophy-and-core-principles)
2. [The Two-Repo Pattern](#2-the-two-repo-pattern)
3. [Git as Message Bus](#3-git-as-message-bus)
4. [The Seven Agents](#4-the-seven-agents)
5. [Model Routing and Token Budget](#5-model-routing-and-token-budget)
6. [The Adapter — Cluster Sidecar](#6-the-adapter--cluster-sidecar)
7. [GitHub Actions — Bidirectional Trigger Layer](#7-github-actions--bidirectional-trigger-layer)
8. [Human Approval Modes](#8-human-approval-modes)
9. [Security Model](#9-security-model)
10. [Memory Architecture](#10-memory-architecture)
11. [Branch Strategy and Lifecycle](#11-branch-strategy-and-lifecycle)
12. [Kill Switch](#12-kill-switch)
13. [Error Recovery and Bug Fix Flow (via Haiku)](#13-error-recovery-and-bug-fix-flow-via-haiku)
14. [Exploration Mode and Literature References](#14-exploration-mode-and-literature-references)
15. [N_EXPERIMENT_TRIAL Guard Rail](#15-n_experiment_trial-guard-rail)
16. [CI/CD for the Brain Itself](#16-cicd-for-the-brain-itself)
17. [Complete File Map](#17-complete-file-map)
18. [Build Order](#18-build-order)
19. [What Was Not Included (Deliberately)](#19-what-was-not-included-deliberately)

---

## 1. Philosophy and Core Principles

**AutoPhD is a research team of agents that attaches to any experiment repository and runs hypothesis-driven experiments methodically.**

It is not a code generator. It is not a paper writer. It is a scientific orchestration layer that:
- Maintains the scientific integrity of experiments (theory pre-checks, verdict framework)
- Keeps costs controlled (budget-first invocation, cheap model routing)
- Keeps humans in control (approval gates, kill switch, traceable audit trail)
- Works with cluster-based experiments (git as the communication bus)

### Core Principles

| Principle | Implication |
|-----------|------------|
| **Git as message bus** | No direct cluster communication. Everything via commits on a shared branch. |
| **Event-driven agents** | GitHub Actions is the trigger. Agents never poll. Zero idle tokens. |
| **Budget before action** | Every agent invocation checks `budget.json` first. Over budget = no call. |
| **Human-in-loop by default** | `approval_required: true` is the default. Autonomous is opt-in. |
| **One adapter file** | Only one file (`autoPhD_adapter.py`) is added to the experiment repo. |
| **Security first** | All external inputs are scanned. All code changes require Security Agent sign-off. |
| **Memory in two tiers** | Experiment: `agent_comms.jsonl` (always) + Mem0 (semantic). Dev: `.memories/` (episodic + semantic). |
| **Hypothesis file drives everything** | Health thresholds, stopping rules, allowed paths — all come from the hypothesis. |

---

## 2. The Two-Repo Pattern

```
autoPhD/                    ← THE BRAIN (this repo)
│
├── team/                        ← Agent behavioral contracts (markdown + frontmatter)
├── templates/                   ← Files generated into the experiment repo
├── cursor_agents/               ← Implementation specs for building this system
├── memory/                      ← Mem0 integration
├── brain/                       ← Python runtime (invoke.py, budget.py, connect.py)
└── projects/{project}/          ← Per-project research files (hypotheses, history)
    ├── project.md
    ├── hypotheses/H{NNN}.md
    ├── history/
    └── context/

experiment-repo/                 ← THE BODY (any experiment repo, on GitHub)
│   (the user's existing code, untouched except for .autoPhD/)
│
├── .autoPhD/               ← Brain writes here (on the experiment branch)
│   ├── config.json              ← Main Agent writes once at setup
│   ├── state.md                 ← Append-only audit trail
│   ├── budget.json              ← Token/trial counters (updated every agent call)
│   ├── agent_comms.jsonl        ← Machine-readable agent exchange log
│   ├── agent_dialogue.md        ← Human-readable agent conversation
│   ├── interrupt.json           ← Brain writes to interrupt/kill experiment
│   ├── heartbeat.json           ← Adapter commits this on health events
│   ├── monitor_report.md        ← Monitor Agent writes
│   ├── final_report.md          ← Analyst Agent writes
│   ├── PENDING_APPROVAL.md      ← Written when human approval is needed
│   ├── NOTIFICATION.md          ← Written when human attention is needed
│   └── bug_report.md            ← Bug Analyst Agent writes
│
├── autoPhD_adapter.py      ← THE ADAPTER (generated, one file)
├── health_checker.py            ← Generated from health_thresholds.md
└── .github/workflows/
    ├── brain_listen.yml         ← Fires on adapter commits (cluster→brain)
    └── experiment_resume.yml    ← Fires on brain commits (brain→cluster notification)
```

**Key invariant**: The brain never touches the experiment repo's source code directly except through the Experiment Agent writing to `allowed_paths`. The adapter is the only brain-authored file at the repo root.

---

## 3. Git as Message Bus

### Cluster → Brain (reporting)

The adapter runs `health_checker.py` on a background thread continuously. When health changes:

```
health_checker detects bad health
  → writes .autoPhD/heartbeat.json
  → git add .autoPhD/heartbeat.json
  → git commit -m "health: {GREEN|YELLOW|RED} iter={n}/{budget} {flag}"
  → git push origin exp/{branch}
  → GitHub Action brain_listen.yml fires
  → brain/invoke.py is called with the event
```

Commit message format (machine-parseable):
```
health: RED iter=87/200 nan_in_seed_7
health: YELLOW iter=80/200 plateau_detected
health: GREEN iter=50/200
complete: iter=200/200 status=success
complete: iter=143/200 status=interrupted
```

### Brain → Cluster (intervention)

When the brain needs to intervene:
```
brain writes .autoPhD/interrupt.json or config.json update
  → git add .autoPhD/
  → git commit -m "brain: interrupt reason=plateau_unresolved"
  → git push origin exp/{branch}
  → GitHub Action experiment_resume.yml fires (notification only)
  → adapter's background thread does git pull every 30s
  → adapter sees interrupt.json → signals main process to stop cleanly
```

### The Bidirectional Contract

| Direction | Trigger | Actor | File Changed |
|-----------|---------|-------|--------------|
| Cluster → Brain | `git push` by adapter | `brain_listen.yml` | `heartbeat.json`, `completion_flag.json` |
| Brain → Cluster | `git push` by brain | `experiment_resume.yml` | `interrupt.json`, `config.json`, `NOTIFICATION.md` |

The Actions do NOT trigger each other in a loop. `brain_listen.yml` only fires on adapter-committed files. `experiment_resume.yml` only fires on brain-committed files. The commit author is the filter.

---

## 4. The Seven Agents

### Agent Hierarchy

```
Human
│
├── Main Agent [OPUS]              ← Scientific orchestrator + Mathematician
│   │
│   ├── Experiment Agent [HAIKU]   ← Writes experiment code directly via API, runs smoke test
│   │
│   ├── Bug Fixer Agent [HAIKU]    ← Reads monitor report + src, writes fix directly, runs smoke test
│   │
│   ├── Monitor Agent [HAIKU]      ← Live log reader + anomaly detector
│   │
│   ├── Analyst Agent [SONNET]     ← Post-run statistician
│   │
│   └── Summarizer Agent [GEMINI]  ← State writer + memory packager
│
└── Security Agent [HAIKU]         ← Injection scanner for external content (web + references)
```

**Key design**: Haiku agents write code directly via the Claude API — no Cursor, no human presence needed for code changes. Sonnet is reserved for statistical analysis requiring extended thinking. Opus is reserved for scientific judgment only. Code safety is enforced by inline Python patterns in `brain/security.py` before any commit.

### Agent Summary Table

| Agent | Model | Thinking | Max Input | Max Output | Invoked By | Full Spec |
|-------|-------|----------|-----------|------------|------------|-----------|
| Main Agent | claude-opus-4-6 | extended | summaries only (~4k) | 2k | GitHub Actions (events) | team/main_agent.md |
| Monitor Agent | claude-haiku-4-5-20251001 | off | heartbeat + 50 log rows (~3k) | 1.5k | brain_listen.yml | team/monitor_agent.md |
| Analyst Agent | claude-sonnet-4-6 | budget=5000 | full log + hypothesis (~20k) | 4k | brain_listen.yml on completion | team/analyst_agent.md |
| Security Agent | claude-haiku-4-5-20251001 | off | web/ref content (~4k) | 1k | After any WebFetch/WebSearch | team/security_agent.md |
| Summarizer Agent | gemini-3.1-flash-lite-preview | off | state + reports (~6k) | 2k | After Analyst + after verdict | team/summarizer_agent.md |
| Experiment Agent | claude-haiku-4-5-20251001 | off | instruction + hypothesis + src files (~8k) | 4k | Main Agent instruction | team/experiment_agent.md |
| Bug Fixer Agent | claude-haiku-4-5-20251001 | off | monitor report + src file (~6k) | 3k | On code failure (RED) | team/bug_analyst_agent.md |

### Communication Protocol

All agents communicate through **files on the experiment branch**, not through direct API calls to each other.

```
Main Agent writes:        .autoPhD/state.md (INSTRUCTION FOR ... block)
Subagent reads:           .autoPhD/state.md
Subagent writes:          .autoPhD/{report}.md
Main Agent reads next:    .autoPhD/{report}.md on next invocation
```

No agent invokes another agent directly. The GitHub Action invocation system is the only entry point.

---

## 5. Model Routing and Token Budget

### Model Assignment Rationale

```
Task requires deep scientific reasoning or theorem checking → Claude Opus 4.6
Task requires code writing or statistical analysis          → Claude Sonnet 4.6
Task is template-filling, writing, or summarization        → Gemini Flash (free tier)
```

### Task → Model Routing Table

| Task | Model | Reason |
|------|-------|--------|
| Hypothesis verdict | claude-opus-4-6 | Scientific judgment, theorem reasoning |
| Theory pre/post check | claude-opus-4-6 | Mathematical reasoning required |
| Config change decision | claude-opus-4-6 | Scientific decision |
| Interrupt/continue decision | claude-opus-4-6 | Risk assessment |
| References reading + insight extraction | claude-opus-4-6 | Scientific interpretation of literature |
| Statistical analysis | claude-sonnet-4-6 + thinking | Computation + interpretation, needs reasoning |
| References fetch for statistics method | claude-sonnet-4-6 | Understanding statistical methods from papers |
| Log analysis, anomaly detection | claude-haiku-4-5 | Pattern recognition, fast, cheap |
| Experiment code writing | claude-haiku-4-5 | Code writing, fully automated via API |
| Bug diagnosis + fix writing | claude-haiku-4-5 | Pattern matching from monitor report |
| Injection scan (web/reference content) | claude-haiku-4-5 | Pattern matching |
| Code safety check (allowed paths, secrets) | Python inline (brain/security.py) | Zero LLM cost — pure regex patterns |
| PR creation | Python (brain/git_utils.py) | Zero LLM cost — gh CLI call |
| History condensation | gemini-3.1-flash-lite-preview | Summarization, free tier |
| Artifact bundle writing | gemini-3.1-flash-lite-preview | Structured summarization, free tier |

### Budget File Structure

`.autoPhD/budget.json` — checked before every agent invocation:

```json
{
  "run_id": "exp-H001-20260317-143022",
  "limits": {
    "n_experiment_trials": 5,
    "max_agent_calls_total": 50,
    "max_cost_usd": 10.00,
    "max_tokens_by_agent": {
      "main": 100000,
      "experiment": 40000,
      "monitor": 40000,
      "analyst": 80000,
      "summarizer": 20000,
      "security": 20000,
      "bug_analyst": 30000
    }
  },
  "used": {
    "trials": 0,
    "agent_calls": 0,
    "cost_usd": 0.0,
    "tokens_by_agent": {
      "main": 0,
      "experiment": 0,
      "monitor": 0,
      "analyst": 0,
      "summarizer": 0,
      "security": 0,
      "bug_analyst": 0
    }
  },
  "last_updated": "2026-03-17T14:23:01Z"
}
```

### Budget Enforcement

`brain/budget.py` runs before every agent invocation:

```python
def check_budget(agent_name: str, estimated_tokens: int) -> BudgetCheckResult:
    budget = load_budget()

    # Check per-agent token limit
    if budget.used.tokens_by_agent[agent_name] + estimated_tokens > budget.limits.max_tokens_by_agent[agent_name]:
        return BudgetCheckResult.BLOCKED, f"Agent {agent_name} token limit reached"

    # Check total cost
    estimated_cost = estimate_cost(agent_name, estimated_tokens)
    if budget.used.cost_usd + estimated_cost > budget.limits.max_cost_usd:
        return BudgetCheckResult.BLOCKED, f"Total cost limit ${budget.limits.max_cost_usd} would be exceeded"

    # Check trial limit (only for experiment relaunches)
    if agent_name == "experiment" and is_relaunch:
        if budget.used.trials >= budget.limits.n_experiment_trials:
            return BudgetCheckResult.BLOCKED, f"Max experiment trials ({budget.limits.n_experiment_trials}) reached"

    return BudgetCheckResult.OK, "Within budget"
```

When budget is exceeded: write `BUDGET_EXHAUSTED.md` to `.autoPhD/`, write `NOTIFICATION.md`, stop all agent activity. Human must review and optionally increase limits.

---

## 6. The Adapter — Cluster Sidecar

`autoPhD_adapter.py` is the **only file added to the experiment repo**. It does exactly four things:

### 1. Wraps run_experiment.py as a subprocess
```python
proc = subprocess.Popen(["python", run_experiment_file, config_path])
```

### 2. Runs health_checker.py in a background thread
```python
health_thread = threading.Thread(target=run_health_loop, daemon=True)
health_thread.start()
```

The health checker loop:
- Reads `log.csv` and `diagnostics.csv` continuously (using file watchdog or polling)
- Runs the stopping rules from `health_checker.py` (generated from health_thresholds.md)
- On any health change: writes `heartbeat.json`, git commits, git pushes
- Checks for `interrupt.json` via `git pull` every 30 seconds
- If `interrupt.json` found: writes `/tmp/{run-id}.interrupt` (local signal file)

### 3. Signals the experiment to stop cleanly
```python
# In main process, experiment polls this local file
while True:
    if Path(f"/tmp/{run_id}.interrupt").exists():
        flush_and_exit()
        break
```

The main experiment process **never touches git**. Only the adapter thread does.

### 4. Commits the initial "started" commit on first run
```python
# When adapter starts:
git_commit(".autoPhD/state.md", f"experiment: started {run_id} H{hypothesis_id}")
git_push()
```

This "started" commit is what triggers the `brain_listen.yml` action for the first time, bootstrapping the pipeline.

### Usage on the cluster

```bash
# Instead of: python run_experiment.py config.json
# Run:        python autoPhD_adapter.py run_experiment.py config.json
```

The adapter passes through all arguments to `run_experiment.py`.

---

## 7. GitHub Actions — Bidirectional Trigger Layer

### brain_listen.yml (Cluster → Brain)

Fires on push to `exp/*` branches when adapter-managed files change.

```yaml
on:
  push:
    branches: ['exp/*']
    paths:
      - '.autoPhD/heartbeat.json'
      - '.autoPhD/completion_flag.json'

jobs:
  invoke-brain:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout experiment repo
        uses: actions/checkout@v4

      - name: Checkout brain repo
        uses: actions/checkout@v4
        with:
          repository: {your-org}/autoPhD
          path: brain
          token: ${{ secrets.AUTORESEARCH_READ_TOKEN }}  # read-only PAT if private

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install brain dependencies
        run: pip install -r brain/requirements.txt

      - name: Run brain invoke
        run: python brain/brain/invoke.py
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          EVENT_TYPE: ${{ github.event.head_commit.message }}
          EXPERIMENT_BRANCH: ${{ github.ref_name }}
```

`brain/invoke.py`:
1. Reads the commit message to determine event type
2. Reads `heartbeat.json` to get health status
3. Checks `budget.json` — abort if over budget
4. Routes to the appropriate agent (monitor / analyst / main)
5. Runs the agent via Claude CLI (`claude --print`) or Gemini API
6. Agent output is written to `.autoPhD/`
7. If action needed (interrupt, config change): commits back and pushes

### experiment_resume.yml (Brain → Cluster notification)

Fires when the brain pushes back to the branch. Does NOT restart the experiment — only creates notifications.

```yaml
on:
  push:
    branches: ['exp/*']
    paths:
      - '.autoPhD/interrupt.json'
      - '.autoPhD/PENDING_APPROVAL.md'
      - '.autoPhD/NOTIFICATION.md'
```

The adapter's background thread picks up `interrupt.json` via its 30-second `git pull` loop.

### Private Repos

For private experiment repos:
- `AUTORESEARCH_READ_TOKEN`: read-only PAT to check out the brain repo
- `ANTHROPIC_API_KEY`: stored as repo secret
- `GEMINI_API_KEY`: stored as repo secret
- `GITHUB_TOKEN`: auto-provided by GitHub Actions (write access to the experiment repo)

The `autoPhD connect` command sets all secrets automatically via `gh secret set`.

---

## 8. Human Approval Modes

### Approval Required (default)

`config.json`: `"approval_mode": "approval_required"`

Flow when brain wants to change code and relaunch:
```
Bug Analyst identifies bug
  → Experiment Agent writes fix to new branch
  → PR Agent creates GitHub PR
  → Brain writes PENDING_APPROVAL.md: "PR #{n} ready for review. Merge to continue."
  → Human reviews PR in GitHub / Cursor / VS Code
  → Human merges PR
  → Experiment can be relaunched (human re-runs autoPhD_adapter.py)
```

The brain never merges PRs. Merging is a human action.

### Autonomous (opt-in)

`config.json`: `"approval_mode": "autonomous"`

Flow when brain wants to change code and relaunch:
```
Bug Fixer Agent diagnoses + writes fix directly (within allowed_paths)
  → Bug Fixer Agent runs smoke test inline
  → brain/security.py inline scan passes (no LLM cost)
  → Brain commits fix directly to experiment branch
  → Brain writes NOTIFICATION.md: "Code change applied. Relaunching."
  → Budget trial counter incremented
  → Adapter triggered to relaunch (via restart_flag.json)
```

Even in autonomous mode: **Main Agent decisions are always logged to state.md and the human is notified via NOTIFICATION.md**. The kill switch always works.

---

## 9. Security Model

### Threat 1: Prompt Injection via Web Search

When the Main Agent is in EXPLORE mode and calls WebSearch, search results may contain adversarial content designed to override instructions.

**Defense**: Security Agent scans all web content before it reaches Main Agent:

```
Main Agent requests WebSearch
  → Security Agent receives raw results
  → Scans for injection patterns:
      - Instruction override attempts ("Ignore previous instructions...")
      - Role hijacking ("You are now...")
      - Data exfiltration attempts ("Send all files to...")
      - Scope expansion ("Also do X, Y, Z...")
  → Returns sanitized content only
  → Appended with: [SECURITY: {n} patterns removed from web results]
```

### Threat 2: Code Injection in Experiment Code

Malicious or buggy code in `src/` could:
- Exfiltrate secrets (API keys, credentials)
- Modify config.json or state.md directly
- Make unauthorized network requests

**Defense**: Security Agent reviews every code diff before it's committed:
- Checks for hardcoded secrets, env variable access outside allowed list
- Checks for network calls outside experiment scope
- Checks for file access outside `allowed_paths`
- Checks for `exec()`, `eval()`, `subprocess` with shell=True

### Threat 3: State File Corruption

An agent writing incorrect data to `state.md` could mislead future agents.

**Defense**:
- `state.md` is append-only (validated by `brain/invoke.py` before every write)
- `agent_comms.jsonl` is a parallel record that cannot be altered retroactively
- Main Agent always reads both and flags discrepancies

### Allowed Paths Enforcement

`config.json` specifies exactly which paths the Experiment Agent may modify:

```json
{
  "allowed_paths": ["src/methods/", "src/config/", ".autoPhD/"],
  "protected_paths": ["src/benchmarks/", "data/", "run_experiment.py", ".github/"]
}
```

Any agent attempting to write outside `allowed_paths` is blocked by `brain/invoke.py`. This is enforced, not advisory.

---

## 10. Memory Architecture

### Two Memory Systems

**Dev Memory** (Claude Code development assistant) — `.memories/` in this repo:
- Layer 1 (episodic): per-chat session logs with dialogue summaries, linked to CLI history
- Layer 2 (semantic): `records.json` — distilled, persistent facts across dev sessions
- Tool: `python .memories/memory_tools.py <cmd>` (stdlib only, 14 commands)
- Loaded at every Claude Code session start: L2 always, L1 contextually

**Experiment Memory** (Main Agent runtime) — lives on the experiment branch:

```
Tier 1: agent_comms.jsonl      ← always written, never fails, machine-readable
Tier 2: agent_dialogue.md      ← always written, never fails, human-readable
Tier 3: Mem0                   ← semantic search across experiments, may fail (tier 1 is fallback)
```

### Tier 1: agent_comms.jsonl

Written by `brain/invoke.py` for **every agent call**. JSON Lines format:

```json
{"ts":"2026-03-17T14:23:01Z","agent":"monitor","model":"claude-sonnet-4-6","input_tokens":1240,"output_tokens":380,"input_files":[".autoPhD/heartbeat.json"],"output_file":".autoPhD/monitor_report.md","duration_ms":4200,"cost_usd":0.0041,"event":"YELLOW_HEARTBEAT","action_taken":"ESCALATE_TO_MAIN","run_id":"exp-H001-20260317","trial":1}
```

This file is the ground truth audit log. It persists on the experiment branch. It can be used to:
- Reconstruct the full agent conversation history
- Debug unexpected behavior
- Audit token and cost spending
- Detect loops or inefficiencies

### Tier 2: agent_dialogue.md

Written by `brain/invoke.py` alongside every agent call. Append-only, human-readable:

```markdown
## 2026-03-17T14:23:01Z — Monitor Agent
**Trigger:** YELLOW heartbeat (iter=80, plateau_detected)
**Model:** claude-sonnet-4-6 | **Tokens:** 1,240 in / 380 out | **Cost:** $0.004 | **Duration:** 4.2s

### Input summary
Heartbeat + last 50 log rows. Regret slope = 0.0001 (threshold -0.001).

### Output
[full monitor_report.md content pasted here]

**Action:** ESCALATE_TO_MAIN_AGENT
---
```

### Tier 3: Mem0

Semantic memory that persists **across experiments and projects**. Used for:
- Recalling past failures with similar configurations
- Pattern matching across hypotheses
- Avoiding repeated mistakes

If Mem0 is unavailable, the system falls back to reading `history/` summaries from the brain repo.

---

## 11. Branch Strategy and Lifecycle

### Branch Naming

```
exp/{project}-{hypothesis_id}-{YYYYMMDD-HHMMSS}

Examples:
exp/riemannian-bo-H001-20260317-143022
exp/riemannian-bo-H001-20260317-161500  ← second trial of same hypothesis
```

The timestamp prevents naming conflicts on relaunches.

### Branch Lifecycle

```
1. Created by:    autoPhD connect (--branch exp/...)
2. Active during: experiment run (adapter + brain both push to it)
3. Complete when: Main Agent writes final verdict to state.md
4. Human action:  Review state.md + final_report.md + agent_dialogue.md
5. Merge:         Squash merge to main with message "H001: SUPPORTED — 18.3% regret"
6. Delete:        Branch deleted after merge (all data preserved in main's squash commit)
```

### When to Merge

Only merge after:
- Main Agent has written a verdict (SUPPORTED / REJECTED / INCONCLUSIVE / PARTIALLY SUPPORTED)
- Human has reviewed `state.md` theory post-check
- Human agrees with the verdict

Do not merge if:
- Experiment is still running
- Budget is exhausted without a verdict
- `PENDING_APPROVAL.md` exists (approval outstanding)

---

## 12. Kill Switch

### CLI Command

```bash
autoPhD kill \
  --repo https://github.com/you/exp-repo \
  --branch exp/riemannian-bo-H001-20260317-143022 \
  --reason "hypothesis fundamentally flawed"
```

### What it does

1. Writes `.autoPhD/interrupt.json`:
   ```json
   {"interrupt": true, "reason": "user_kill", "kill_permanent": true, "ts": "..."}
   ```
2. Commits and pushes to the experiment branch
3. The adapter's background thread sees it within 30 seconds → kills the experiment subprocess
4. Writes a final `state.md` entry: `[KILLED by user — {reason}]`
5. **No further agent calls fire.** `kill_permanent: true` suppresses all subsequent GitHub Actions triggers.

### What it does NOT do

- It does NOT delete the branch (branch is preserved for inspection)
- It does NOT run the Analyst Agent (partial data is not analysed automatically after a user kill)
- It does NOT write a hypothesis verdict (the human decides)

### Emergency kill (if CLI not available)

Go to the experiment repo on GitHub, edit `.autoPhD/interrupt.json` manually and commit with message `"kill: user_emergency"`. The adapter will pick it up within 30 seconds.

---

## 13. Error Recovery and Bug Fix Flow (via Haiku)

### The Problem

The Experiment Agent writes code. Code has bugs. Who fixes them?

**Option A (approval_required mode, default)**: Bug Fixer Agent diagnoses + writes fix → `brain/security.py` scans the diff inline → `brain/git_utils.py` creates a GitHub PR → brain writes `PENDING_APPROVAL.md` → human reviews PR on GitHub → human merges → human re-runs adapter.

**Option B (autonomous mode)**: Bug Fixer Agent diagnoses + writes fix → inline security scan passes → brain commits fix directly → `NOTIFICATION.md` written → budget trial counter incremented → adapter restart signalled.

### Code Change Flow (both modes)

```
Bug detected (RED heartbeat or smoke_test_failed)
  │
  ├── Monitor Agent [Haiku] reads logs → writes monitor_report.md
  │
  ├── Bug Fixer Agent [Haiku] reads:
  │   - monitor_report.md (symptom + Monitor's technical hypothesis)
  │   - heartbeat.json (which flag triggered RED)
  │   - src/{identified file} (the file most likely containing the bug)
  │   → writes the fix directly to src/ (within allowed_paths)
  │   → runs smoke test inline
  │   → writes .autoPhD/bug_completed.json
  │
  ├── brain/security.py scans the diff (inline Python, no LLM cost):
  │   - checks allowed_paths
  │   - checks for secret exposure, unsafe patterns
  │   → PASS or BLOCK
  │
  ├── If PASS:
  │   ├── [approval_required] → brain/git_utils.py creates PR → PENDING_APPROVAL.md
  │   └── [autonomous] → brain commits fix directly → NOTIFICATION.md
  │
  └── If BLOCK:
      → brain writes security_review.md with reason → Main Agent decides
```

### Snapshot Before Any Code Change

`brain/invoke.py` creates a git snapshot before invoking any code-writing agent:
```python
git_commit_and_push(
    files=["src/"],
    message=f"snapshot: pre-agent-modification trial-{trial_n}"
)
```

This snapshot allows recovery if the fix breaks things worse. The snapshot is never automatically reverted — the human decides whether to roll back.

### Human Review of PRs

Brain creates standard GitHub PRs. The human can review them in GitHub's web UI, VS Code, or any editor.

```bash
# To review a PR locally:
git fetch origin {fix-branch}
git checkout {fix-branch}
# Review changes, test locally, then merge on GitHub
```

---

## 14. Exploration Mode and Literature References

### User-provided references (always available — GUIDED and EXPLORE)

Any hypothesis file may include a `## References` section with paper URLs, DOIs, or GitHub repo links. When present, **two agents** may fetch these:

1. **Main Agent** (Opus): Fetches references for scientific insight — understanding the mechanism, checking theorem applicability, informing the theory pre-check. Uses WebFetch (not WebSearch).
2. **Analyst Agent** (Sonnet): Fetches references that explain *how to compute* the statistical method specified in the `## Statistical Analysis Plan`. Does not fetch for scientific interpretation.

Both agents route fetched content through the Security Agent (Haiku) for injection scanning before using it.

This is a **pull** model — the user decides which references are relevant by including them in the hypothesis file. The agents do not search for references autonomously.

### Default: GUIDED

The brain works only with:
- The hypothesis file the user wrote
- User-provided references in `## References` (fetched via WebFetch)
- The theory in `context/theorems.md`
- The prior work in `context/prior_work.md`
- Results from previous experiments (via Mem0 + history/)

No WebSearch. No autonomous hypothesis generation.

### Opt-in: EXPLORE

Activated by setting `exploration_mode: EXPLORE` in the hypothesis file **and committing that change yourself**. (The brain verifies the commit author to prevent self-activation.)

In EXPLORE mode, the Main Agent may:
- Call WebSearch with a specific query
- But **must** write a justification to `state.md` first:
  ```
  ### {ts} — Web Search Request [Main Agent]
  Query: "{specific query}"
  Justification: "The theorem cited in H001 §3 has a gap: it assumes compactness of X,
  but our benchmark does not satisfy this. Searching for whether this assumption can be relaxed."
  Expected result: "A paper or proof that either extends the theorem or shows a counterexample."
  ```
- All search results are passed through the Security Agent before reaching Main Agent
- Search results are stored in `context/web_search_log.md` for auditability

The Main Agent may NOT in EXPLORE mode:
- Autonomously propose and launch new hypotheses without human approval
- Modify `context/theorems.md` (that is a human-curated file)
- Call WebSearch more than 3 times per session

---

## 15. N_EXPERIMENT_TRIAL Guard Rail

Each run of `autoPhD_adapter.py` counts as one **trial**. The budget limits `n_experiment_trials`.

Trial counter increments when:
- The adapter starts a new run (including relaunches after interrupt)
- A bug fix is applied and the experiment re-runs (autonomous mode)

Trial counter does NOT increment for:
- Early success stops (experiment finished with positive result)
- Monitor Agent check-ins (not a trial)
- Agent-only sessions (Main Agent verdict, Analyst analysis)

When `trials >= n_experiment_trials`:
- `brain/budget.py` blocks all further agent invocations
- Writes `BUDGET_EXHAUSTED.md`: "Max trials ({n}) reached for H{NNN}. Human review required."
- Human must either increase `n_experiment_trials` in `config.json` or close the hypothesis

---

## 16. CI/CD for the Brain Itself

The AutoPhD repo should have its own CI:

`.github/workflows/validate_brain.yml`:
```yaml
on: [push, pull_request]
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Validate agent frontmatter
        run: python ci/validate_agent_frontmatter.py
        # Checks every team/*.md has valid frontmatter (agent, model, reads, writes fields)

      - name: Validate template files
        run: python ci/validate_templates.py
        # Checks templates are syntactically valid

      - name: Run brain unit tests
        run: python -m pytest tests/ -v
        # Tests: budget.py, invoke.py routing, security scanner patterns

      - name: Lint cursor_agents specs
        run: python ci/validate_cursor_agents.py
        # Checks cursor_agent files have required sections
```

---

## 17. Complete File Map

### Brain Repo (autoPhD/)

```
autoPhD/
├── ARCHITECTURE.md          ← This file
├── CLAUDE.md                ← Main Agent boot instructions
├── README.md                ← Human-facing overview
│
├── brain/
│   ├── connect.py           ← `autoPhD connect` CLI command
│   ├── invoke.py            ← Routes events to agents, checks budget, logs to agent_comms.jsonl
│   ├── budget.py            ← Budget enforcement before every call
│   ├── security.py          ← Injection scanner, code safety checker
│   ├── git_utils.py         ← Git operations (commit, push, pull, PR creation)
│   └── watched_repos.json   ← Registry of connected experiment repos
│
├── team/
│   ├── roles.md             ← Quick reference (all 7 agents)
│   ├── main_agent.md        ← claude-opus-4-6
│   ├── monitor_agent.md     ← claude-haiku-4-5
│   ├── analyst_agent.md     ← claude-sonnet-4-6 + thinking
│   ├── experiment_agent.md  ← claude-haiku-4-5
│   ├── bug_analyst_agent.md ← claude-haiku-4-5
│   ├── security_agent.md    ← claude-haiku-4-5
│   └── summarizer_agent.md  ← gemini-3.1-flash-lite-preview
│
├── templates/
│   ├── hypothesis_template.md         ← User fills this in (includes ## References section)
│   ├── health_thresholds_template.md  ← User fills this in
│   ├── state_template.md
│   ├── log_schema.md
│   ├── heartbeat_schema.md
│   ├── config_template.json           ← Full config schema with all fields
│   ├── autoPhD_adapter.py.template ← The cluster sidecar
│   ├── health_checker.py.j2           ← Jinja2 template, filled from health_thresholds
│   ├── brain_listen.yml               ← GitHub Actions (cluster→brain)
│   └── experiment_resume.yml          ← GitHub Actions (brain→cluster)
│
├── .memories/                         ← Dev memory system (for Claude Code assistant)
│   ├── memory_tools.py                ← CLI tool (stdlib, 14 commands)
│   ├── index.json                     ← Layer 1: chat session index
│   ├── records.json                   ← Layer 2: persistent semantic facts
│   ├── json/                          ← Per-session dialogue JSON files
│   └── markdowns/                     ← Per-session human-readable summaries
│
├── memories/                          ← Legacy flat session logs (migrated to .memories/)
│
├── memory/
│   ├── memory_schema.md
│   ├── memory_bridge.py               ← Mem0 read/write interface
│   └── mem0_config_template.json
│
├── cursor_agents/                     ← Build specs for Claude Code to implement brain/ components
│   ├── 00_overview.md                 ← Build order and overview
│   ├── 01_build_budget_guard.md
│   ├── 02_build_invoke_layer.md
│   ├── 03_build_adapter_template.md
│   ├── 04_build_health_checker_gen.md
│   ├── 05_build_github_actions.md
│   ├── 06_build_connect_cli.md
│   └── 07_build_test_suite.md
│
├── tests/
│   ├── test_budget.py
│   ├── test_invoke_routing.py
│   ├── test_security_scanner.py
│   └── test_adapter_template.py
│
├── ci/
│   ├── validate_agent_frontmatter.py
│   ├── validate_templates.py
│   └── validate_cursor_agents.py
│
└── projects/
    └── {project}/
        ├── project.md
        ├── health_thresholds.md    ← Project-specific thresholds
        ├── hypotheses/H{NNN}.md
        ├── history/
        └── context/
            ├── theorems.md
            └── prior_work.md
```

### What Gets Created in the Experiment Repo (on `autoPhD connect`)

```
experiment-repo/  (on exp branch)
├── autoPhD_adapter.py    ← copied from template, parameterized
├── health_checker.py          ← generated from health_thresholds.md
├── .autoPhD/
│   ├── config.json            ← Main Agent writes
│   ├── state.md               ← initial state
│   ├── budget.json            ← initial budget
│   ├── agent_comms.jsonl      ← empty, ready for writes
│   └── agent_dialogue.md      ← empty, ready for writes
└── .github/workflows/
    ├── brain_listen.yml
    └── experiment_resume.yml
```

---

## 18. Build Order

Build these components in order. Each step is independently testable.

```
Step 1: templates/health_thresholds_template.md
        → The user fills this before anything runs. Define it first.

Step 2: templates/config_template.json
        → Defines all fields the system uses. Everything else reads from this.

Step 3: brain/budget.py
        → Must exist before any agent calls. Nothing runs without budget check.
        → Test: unit tests in tests/test_budget.py

Step 4: brain/security.py
        → Must exist before any web searches or code changes.
        → Test: unit tests with known injection patterns

Step 5: team/*.md frontmatter
        → Add machine-readable frontmatter to all agent files.
        → Test: ci/validate_agent_frontmatter.py

Step 6: brain/invoke.py
        → Routes events to agents. Reads frontmatter, calls claude/gemini, logs to agent_comms.
        → Test: tests/test_invoke_routing.py

Step 7: templates/autoPhD_adapter.py.template
        → The cluster sidecar. Can be tested standalone.
        → Test: tests/test_adapter_template.py

Step 8: templates/health_checker.py.j2
        → Generated from health_thresholds. Test with sample thresholds file.

Step 9: templates/brain_listen.yml + experiment_resume.yml
        → GitHub Actions. Test with a test repo and simulated pushes.

Step 10: brain/git_utils.py
         → Git operations used by invoke.py. Test with a local bare repo.

Step 11: brain/connect.py
         → The CLI command. Integration test end-to-end.

Step 12: CI for autoPhD itself
         → .github/workflows/validate_brain.yml
```

---

## 19. What Was Not Included (Deliberately)

| Feature | Reason not included |
|---------|-------------------|
| SLURM templates | User manages cluster submission. Too cluster-specific. |
| Automatic paper writing | Out of scope. AutoPhD stops at hypothesis verdict. |
| Literature scraping | Out of scope. User provides context/prior_work.md. |
| Multi-hypothesis parallelism | Too complex for v1. One hypothesis at a time. |
| Dashboard / UI | Out of scope. GitHub is the UI. |
| Automatic hypothesis generation | Only in EXPLORE mode, still requires human approval to launch. |
| Cursor integration | Removed. All code writing done by Haiku agents via API — fully automated. |
| Email/Slack notifications | NOTIFICATION.md + GitHub notifications are sufficient for v1. |

These can be added in v2 without changing the core architecture.
