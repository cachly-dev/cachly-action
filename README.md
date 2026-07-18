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

PR risk scan. Sends the PR title and body to the Brain's `/scan` API, which matches them against your failure lessons, then posts a comment with a risk score (0–100) and a list of predicted failure patterns. Run this on `pull_request: types: [opened, synchronize]` before CI runs so reviewers see the risk upfront.

### `predict` — sticky "your Brain would have warned you" comment

Like `scan`, but built for repeat pushes: it matches the PR title, body **and
changed file paths** against your Brain's failure lessons and maintains **one**
sticky PR comment (marker `<!-- cachly-brain-predict -->`) that is created once
and updated in place on every push — it never spams a new comment per commit.

- Predictions above `predict-min-confidence` → the comment lists each pattern
  with severity, confidence and the known fix.
- No predictions → **no comment at all**. If an earlier push left a warning
  comment, it is updated and collapsed to "no known risks" instead of deleted.

Enable it standalone with `mode: predict`, or add `predict-comment: 'true'` to
an existing `scan` job (the sticky comment then replaces the legacy per-push
scan comment). See the copy-paste snippet below.

### `confirm`

Reports the CI outcome back to the Brain at the end of the pipeline so lesson
confidences self-calibrate. On `ci-job-status: failure` it **also records the
failure as a Brain lesson automatically** (topic = first `ci-topics` entry,
outcome `failure`, linked to the failing run) — so the next PR that touches the
same area gets a "your Brain would have warned you" prediction. Both calls are
best-effort: missing secrets or an unreachable API never fail your CI.

### `hygiene`

Weekly Brain sweep. Archives stale lessons, flags provisional knowledge, and removes orphaned entries. Schedule via `workflow_dispatch` or a weekly cron — keeps Brain quality high over time.

## GitLab CI/CD

On GitLab? The same Brain and the same closed-loop self-calibration are
available for the `learn`, `scan`, `confirm` and `hygiene` modes via the GitLab template at
[`templates/cachly.gitlab-ci.yml`](templates/cachly.gitlab-ci.yml)
(`setup`, `predict` and `hygiene` are GitHub-Action-only for now). Add to your
`.gitlab-ci.yml`:

```yaml
include:
  - remote: 'https://raw.githubusercontent.com/cachly-dev/cachly-action/main/templates/cachly.gitlab-ci.yml'

variables:
  CACHLY_INSTANCE_ID: "<your-brain-uuid>"

cachly-learn:
  extends: .cachly_learn
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

cachly-scan:
  extends: .cachly_scan
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

cachly-confirm:
  extends: .cachly_confirm
  when: always          # run even if earlier jobs failed, so failures are reported
  variables:
    CACHLY_CI_TOPICS: "auth:jwt,deploy:k8s"
```

Set `CACHLY_API_KEY` (masked) in **Settings → CI/CD → Variables**. Outcomes are
reported to the Brain as source `gitlab_ci`. On a failed pipeline the `confirm`
job records the failure as a Brain lesson automatically (same as the GitHub
Action). The Cachly VS Code and JetBrains plugins auto-detect a GitLab `origin`
and scaffold this for you on setup.

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
| `mode` | ❌ | `setup` | `setup` (generate config + index), `learn` (auto-learn from recent commits), `scan` (PR risk scan), `predict` (sticky PR prediction comment), `hygiene` (weekly Brain sweep), or `confirm` (report CI outcome back to the Brain) |
| `learn-max-commits` | ❌ | `50` | In `learn` mode: how many recent commits to learn from |
| `pr-number` | ❌ | — | PR number (required for `scan` / `predict` modes) |
| `pr-title` | ❌ | — | PR title passed to risk scan / prediction |
| `pr-body` | ❌ | — | PR body / description passed to risk scan / prediction |
| `scan-top-k` | ❌ | `7` | In `scan` / `predict` mode: number of Brain lessons to consider |
| `scan-post-comment` | ❌ | `true` | In `scan` mode: whether to post a PR comment with the risk score |
| `predict-comment` | ❌ | `false` | Maintain ONE sticky Brain prediction comment on the PR (create-or-update, never spams per push). Implied by `mode: predict` |
| `github-token` | ❌ | workflow token | In `predict` mode: token to read PR changed files and create/update the sticky comment (`pull-requests: write`) |
| `predict-min-confidence` | ❌ | `0.5` | In `predict` mode: minimum confidence (0–1) for a prediction to appear in the comment |
| `ci-job-status` | ❌ | — | In `confirm` mode: `success`, `failure` or `cancelled`. On `failure` a Brain lesson is recorded automatically |
| `ci-topics` | ❌ | — | In `confirm` mode: comma-separated Brain topics touched by this CI run |
| `ci-scan-topics` | ❌ | — | In `confirm` mode: topics a previous scan predicted would fail (false-positive detection) |

## Outputs

| Output | Description |
|--------|-------------|
| `indexed-files` | Number of files indexed into the Brain |
| `risk-score` | Risk score 0–100 produced by `scan` mode (0 = low risk, 100 = very high) |
| `failures-json` | JSON array of predicted failure patterns from `scan` mode |
| `predict-warnings` | Number of Brain warnings surfaced by `predict` mode (0 = comment stays silent) |
| `ci-updated` | Number of lesson confidences updated by `confirm` mode |

---

## 🧠 "Your Brain would have warned you" — sticky PR prediction comment

Copy-paste this workflow and every pull request gets checked against your
team's failure history. If the Brain knows a matching pattern, it posts **one**
comment like:

> ## 🧠 Cachly Brain: 2 Warnungen für diesen PR
> - 🔴 `deploy:k8s` — **90% confidence** — Rollout stuck: readinessProbe too strict → Fix: increase failureThreshold to 10
> - 🟡 `web:hydration` — **65% confidence** — typeof window in useState initializer → Fix: useState(safeDefault) + useEffect

The comment is updated in place on every push (never one comment per commit),
and if the risks disappear it collapses to "no known risks". PRs with no
matching patterns get **no comment at all**.

```yaml
name: Brain predict
on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write   # needed to create/update the sticky comment

jobs:
  predict:
    runs-on: ubuntu-latest
    steps:
      - uses: cachly-dev/cachly-action@v1
        with:
          mode: predict
          instance-id: ${{ secrets.CACHLY_INSTANCE_ID }}
          api-key: ${{ secrets.CACHLY_API_KEY }}
          pr-number: ${{ github.event.pull_request.number }}
          pr-title: ${{ github.event.pull_request.title }}
          pr-body: ${{ github.event.pull_request.body }}
          # github-token defaults to the workflow token — override for a bot identity:
          # github-token: ${{ secrets.BOT_TOKEN }}
          # predict-min-confidence: '0.5'
```

No checkout step needed — changed files are read via the GitHub API. Missing
secrets or an unreachable Brain never fail the job. Close the loop by feeding
CI results back with `mode: confirm` (`ci-job-status: ${{ job.status }}`): red
pipelines become lessons automatically, so the *next* PR touching the same
area gets warned.

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
