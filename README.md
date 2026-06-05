# cachly-brain-setup

GitHub Action to auto-configure AI Brain memory for any repo.

## Usage

```yaml
name: Setup Cachly Brain
on:
  push:
    branches: [main]

jobs:
  setup-brain:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: cachly-dev/cachly-action@v1
        with:
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
          project-description: 'My awesome project'

      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'chore: configure Cachly AI Brain'
          file_pattern: '.github/copilot-instructions.md .mcp.json'
```

## Modes

### `setup` (default)

Generates `.github/copilot-instructions.md` and `.mcp.json`, then indexes your source files into the Brain. Run on every push to `main` to keep the index fresh.

### `learn`

Auto-learns from recent commits and PR metadata. Wire this to `pull_request: types: [closed]` so your Brain grows on every merge — no manual `learn_from_attempts` calls needed.

### `scan`

PR risk scan. Calls `brain_plan` + `smart_recall` against the PR diff and posts a comment with a risk score (0–100) and a list of predicted failure patterns. Run this on `pull_request: types: [opened, synchronize]` before CI runs so reviewers see the risk upfront.

### `hygiene`

Weekly Brain sweep. Archives stale lessons, flags provisional knowledge, and removes orphaned entries. Schedule via `workflow_dispatch` or a weekly cron — keeps Brain quality high over time.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `instance-id` | ✅ | — | Brain instance UUID (from cachly.dev → Dashboard → Brain) |
| `api-key` | ✅ | — | API key `cky_live_...` (use GitHub Secrets!) |
| `api-url` | ❌ | `https://api.cachly.dev` | API base URL |
| `embed-provider` | ❌ | `openai` | Embedding provider |
| `project-description` | ❌ | repo name | Short project description |
| `index-project` | ❌ | `true` | Index source files into the Brain semantic cache |
| `index-max-files` | ❌ | `500` | Max files to index per run |
| `mode` | ❌ | `setup` | `setup` (generate config + index), `learn` (auto-learn from recent commits), `scan` (PR risk scan), or `hygiene` (weekly Brain sweep) |
| `learn-max-commits` | ❌ | `50` | In `learn` mode: how many recent commits to learn from |
| `pr-number` | ❌ | — | PR number (required for `scan` mode) |
| `pr-title` | ❌ | — | PR title passed to risk scan |
| `pr-body` | ❌ | — | PR body / description passed to risk scan |
| `scan-top-k` | ❌ | `10` | In `scan` mode: number of Brain lessons to consider |
| `scan-post-comment` | ❌ | `true` | In `scan` mode: whether to post a PR comment with the risk score |

## Outputs

| Output | Description |
|--------|-------------|
| `indexed-files` | Number of files indexed into the Brain |
| `risk-score` | Risk score 0–100 produced by `scan` mode (0 = low risk, 100 = very high) |
| `failures-json` | JSON array of predicted failure patterns from `scan` mode |

---

## Full workflow example

All four modes wired together in one CI file:

```yaml
name: Cachly Brain

on:
  push:
    branches: [main]
  pull_request:
    types: [opened, synchronize, closed]
  schedule:
    - cron: '0 3 * * 1'   # weekly hygiene — every Monday at 03:00 UTC
  workflow_dispatch:

jobs:
  # ── setup: keep the Brain index fresh on every push to main ──────────
  setup:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: setup
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
          project-description: 'My awesome project'
      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: 'chore: configure Cachly AI Brain'
          file_pattern: '.github/copilot-instructions.md .mcp.json'

  # ── scan: post PR risk comment before CI runs ─────────────────────────
  scan:
    if: github.event_name == 'pull_request' && github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: scan
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
          pr-number: ${{ github.event.pull_request.number }}
          pr-title: ${{ github.event.pull_request.title }}
          pr-body: ${{ github.event.pull_request.body }}
          scan-top-k: 10
          scan-post-comment: 'true'

  # ── learn: auto-learn from every merged PR ────────────────────────────
  learn:
    if: github.event_name == 'pull_request' && github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 50
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: learn
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}

  # ── hygiene: weekly Brain sweep ───────────────────────────────────────
  hygiene:
    if: github.event_name == 'schedule' || github.event_name == 'workflow_dispatch'
    runs-on: ubuntu-latest
    steps:
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: hygiene
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
```

---

## 🧠 Auto-learn from every merged PR (zero manual work)

Add this workflow and your Brain grows automatically on every PR merge —
each meaningful commit becomes a lesson your AI recalls in future sessions:

```yaml
name: Brain — learn from merged PRs
on:
  pull_request:
    types: [closed]

jobs:
  learn:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 50   # must be >= learn-max-commits for history access
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: learn
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
```

After this, every merged PR teaches your Brain — no `learn_from_attempts`
calls, no manual steps. Your AI arrives pre-briefed on what the team shipped.

## What it generates

- `.github/copilot-instructions.md` — Auto-recall/learn instructions for all AI assistants
- `.mcp.json` — MCP server config (if not present)

After setup, every push automatically indexes your latest source changes into the Brain so your AI assistant can find relevant code with `semantic_search` — no manual re-indexing needed.

---

## 👥 Team Brain — Shared AI Memory for Your Whole Team

One shared instance. Every developer gets smarter every day.

```yaml
# GitHub Actions — share the same Brain instance across the whole team
env:
  CACHLY_INSTANCE_ID: ${{ secrets.CACHLY_INSTANCE_ID }}

# Alice's workflow stores a lesson after a successful deploy:
- name: Store deploy lesson
  run: |
    curl -sX POST https://api.cachly.dev/v1/brain/learn \
      -H "Authorization: Bearer ${{ secrets.CACHLY_API_KEY }}" \
      -d '{"topic":"deploy:k8s-timeout","outcome":"success",
           "what_worked":"Increase readinessProbe.failureThreshold to 10","author":"alice"}'

# Bob's next session-start workflow pulls all team lessons automatically:
- uses: cachly-dev/cachly-action@v1
  with:
    instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
    api-key: ${{ secrets.CACHLY_API_KEY }}
# → "💡 alice solved deploy:k8s-timeout 1d ago: Increase readinessProbe..."
```

Set up a team org at [cachly.dev/teams](https://cachly.dev/teams) — Team €99/mo · 10 seats · Business €299/mo · 50 seats.
