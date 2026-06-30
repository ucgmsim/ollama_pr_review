# Ollama PR Review – Reusable Workflow

This repository contains a reusable *GitHub Actions workflow* that automatically reviews pull requests using a self-hosted runner and the **Ollama Cloud API** (`deepseek-v4-flash:cloud`). The system posts inline comments on PRs, highlighting bugs, security issues, and other code problems.

---

## Architecture
```
┌────────────────────┐     Outbound HTTPS      ┌────────────────────────┐
│ GitHub Actions     │ <────────────────────── │ hypocentre (self-      │
│ (organization)     │ ──────────────────────> │ hosted runner)         │
└────────────────────┘     Poll for jobs       └───────────┬────────────┘
                                                           │
                                                           │ HTTPS
                                                           ▼
                                               ┌─────────────────────────┐
                                               │ Ollama Cloud API        │
                                               │ (deepseek-v4-flash)     │
                                               └─────────────────────────┘
```

- **GitHub Actions** triggers the workflow on PR events.
- **hypocentre** (self-hosted runner) executes the workflow steps.
- **Ollama Cloud API** processes the diff and returns code review comments.
- **GitHub** posts the comments inline on the PR.

---

## Repository Contents

| File | Purpose |
| :--- | :--- |
| `.github/workflows/pr-review.yml` | The reusable workflow that performs the AI review. |
| `README.md` | This documentation. |

---
## Setting on GitHub (Organisation Settings)

1. Go to **Organization Settings** → **Actions** → **Runners** → **New self-hosted runner**.
This gives a set of instructions to execute on Hypocentre - The action runner “hypocentre” was created this way. The instruction contains an auto-generated token to run with “config.sh”. We will be executing provided commands on "Hypocentre".

2. Allow **Public repositories** at **Actions** > **Runner groups** > **Default** > **Repository access** 


## Setting on Hypocentre

Just follow the instruction given by GitHub **Runners** > **Create self-hosted runner** page (step 1 above).  

However, the script ./run.sh will run in foreground and it is not persistent (If Hypocentre re-starts, it is gone). Run run.sh for the initial set it up, and use svc.sh to set up as a persistent service. See below for more details about svc.sh

### 1. Self-Hosted Runner Installation
The runner is installed in `/home/swadmin/actions-runner-org` on `hypocentre`.

**Key files:**
- `./run.sh` – runs the runner in foreground (debugging).
- `./svc.sh` – manages the systemd service.
- `./_diag/` – diagnostic logs.
- `./.runner` – registration configuration.

**Runner type:** Organization-level runner for `ucgmsim`.

**Labels:** `self-hosted`, `Linux`, `X64`

### 2. Environment Variables & Secrets

| Secret | Purpose | Where to set |
| :--- | :--- | :--- |
| `OLLAMA_API_KEY` | API key for Ollama Cloud | GitHub organization → Settings → Secrets and variables → Actions |
| `GITHUB_TOKEN` | Provided automatically by GitHub | (Auto-generated) |

### 3. Systemd Service

The runner runs as a systemd service:
`actions.runner.ucgmsim.hypocentre.service`


---

## Runner Management Commands

All commands must be run from the runner directory:

```bash
cd ~/actions-runner-org
```

### Service Management (Normal Operation)
| Action	| Command |
| :--- | :--- |
| Start	| `sudo ./svc.sh start` |
| Stop	| `sudo ./svc.sh stop` |
| Status |	`sudo ./svc.sh status` |
| Restart	| `sudo ./svc.sh stop && sudo ./svc.sh start` |
| Install (first time)	| `sudo ./svc.sh install` |
| Uninstall	| `sudo ./svc.sh uninstall` |

### Foreground Mode (Debugging)
Run the runner interactively to see live logs:

```bash
sudo ./svc.sh stop   # stop service first
./run.sh
```
To exit: `Ctrl+C`. Then restart the service:
```bash
sudo ./svc.sh start
```

### Logs & Diagnostics

| What	| Command |
| :--- | :--- |
| Service logs	| `sudo journalctl -u actions.runner.ucgmsim.hypocentre -f` |
| Diagnostic logs	| `tail -f ~/actions-runner-org/_diag/Runner_*.log` |
| Connectivity check	| `./run.sh --check --url https://github.com/ucgmsim --pat <YOUR_PAT>`|

Note: `<YOUR_PAT>` above is the **Personal access tokens** you can generate from https://github.com/settings/tokens 

### Re-registering the Runner
If the runner becomes stale or needs new labels:
```bash
./config.sh remove
./config.sh --url https://github.com/ucgmsim --token <FRESH_TOKEN> --labels self-hosted,Linux,X64
sudo ./svc.sh install
sudo ./svc.sh start
```
Note: The registration token is obtained from:
`https://github.com/ucgmsim` → **Settings** → **Actions** → **Runners** → **New self-hosted runner** → **Linux** → copy the token.

--- 

## Centralized GitHub Actions Workflow
To avoid duplicating `pr-review.yml` across every repository, the workflow has been centralised into this reusable workflow.

### 1. The Reusable Workflow
The reusable workflow is located at: 
```
ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml
```
It contains all the review logic, including:
- Fetching the PR diff.
- Truncating large diffs.
- Crafting a prompt.
- Calling the Ollama Cloud API.
- Posting inline comments.

#### Key features:
- Concurrency: Only one review runs per PR at a time. New commits cancel in-progress reviews.
- Condition: Only runs if the PR is open and from the same repo.
- Diff handling: Truncates large diffs to stay within the model's token limit.
- Inline comments: Posts suggestions directly on the changed lines (with 6 lines of context).

#### Truncation & Model Configuration
The reusable workflow uses the `deepseek-v4-flash:cloud` model with a `MAX_BYTES=700000` truncation limit. This model has a 1M token context window, which comfortably handles diffs up to ~700KB. For models with smaller context windows (e.g., 256K tokens), you would need to reduce MAX_BYTES accordingly.

**Code snippet (from ollama_pr_review/.github/workflows/pr-review.yml):**
```yaml
name: AI Code Review (Reusable)

on:
  workflow_call:
    secrets:
      OLLAMA_API_KEY:
        required: true
        description: 'API key for Ollama Cloud'

concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read
  pull-requests: write
  issues: write

jobs:
  review:
    if: ${{ github.event.pull_request.state == 'open' && github.event.pull_request.head.repo.full_name == github.repository }}
    runs-on: [self-hosted, Linux, X64]
    steps:
      # ... all the review steps ...

      - name: Run Ollama Cloud review
        id: review
        run: |
          # ... setup ...

          DIFF_SIZE=$(wc -c < pr.diff)
          echo "📏 Original diff size: $DIFF_SIZE bytes"
          echo "$DIFF_SIZE" > diff_size.txt

          MAX_BYTES=700000

          if [ "$DIFF_SIZE" -gt "$MAX_BYTES" ]; then
            echo "⚠️ Diff is too large. Truncating to $MAX_BYTES bytes."
            head -c "$MAX_BYTES" pr.diff > pr.diff.truncated
            echo "true" > was_truncated.txt
            mv pr.diff.truncated pr.diff
          else
            echo "✅ Diff size OK"
            echo "false" > was_truncated.txt
          fi

          MODEL_NAME="deepseek-v4-flash:cloud"
          echo "$MODEL_NAME" > model.txt
        # ... retrieves the output from the model and posts comments
```

### 2. The Caller Workflow (Per Repository)
Each repository that wants AI reviews must have a **minimal wrapper workflow** at `.github/workflows/pr-review.yml`:

```yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  call-review:
    uses: ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml@v1.0
    secrets:
      OLLAMA_API_KEY: ${{ secrets.OLLAMA_API_KEY }}
```
#### Automatic Inheritance via the Template Repository
The `ucgmsim/template` repository already contains this caller workflow at `.github/workflows/pr-review.yml`

This means that any new repository created from the template will automatically inherit the AI Code Review workflow. No additional setup is required – the wrapper workflow is pre‑configured and ready to use.

**For existing repositories** : If a repository was created before the template was updated, you can manually add the wrapper workflow by copying the caller workflow file from the template repository.

##### Benefits of this approach:
- Zero setup for new repositories – the AI review is ready from day one.
- Consistency – all repositories start with the same workflow.
- Automatic updates – when the template is updated, new repositories get the latest version.

## Versioning: The Mutable Tag Strategy

### What We Use

The `ollama_pr_review` repository uses mutable tags to distribute updates. The core principle is:
- `@v1.0` is a mutable tag that always points to the latest `v1.0.x` patch release.
- **Caller workflows** pin to @v1.0 (as in `ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml@v1.0`)
- **Patches** are automatically applied; major changes require explicit updates.

### How Caller Workflows Are Pinned
In each repository's `.github/workflows/pr-review.yml`:
```yaml
uses: ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml@v1.0
```
This means all repositories using using @v1.0 **automatically receive** patch updates (e.g., `v1.0.1`, `v1.0.2`) when the tag is moved.

### Maintaining the Mutable Tag
When you make a patch update (bug fix, minor improvement):

#### Patch update (e.g., v1.0.1):
```bash
# 1. Create a static patch tag for the new commit
git tag v1.0.1

# 2. Move the mutable tag to the same commit
git tag -f v1.0

# 3. Push both tags
git push origin v1.0.1
git push --force origin v1.0
```

**Important**: `git tag -f v1.0` forces the `v1.0` tag to point to the current HEAD. Since `v1.0.1` was just created at the same HEAD, `v1.0` now points to the same commit as `v1.0.1`. This is how the mutable tag stays updated.

#### Major Version Update (e.g., v2.0)
If a major change warrants `v2.0`, the process is similar but **requires caller repositories to update their workflows**.
```bash
# Create the static version tag
git tag v2.0.0

# Create the mutable tag pointing to the same commit
git tag -f v2.0

# Push both
git push origin v2.0.0
git push --force origin v2.0
```

**Caller repositories** that wish to use the new major version (and its future patches, e.g., `v2.0.1`, `v2.0.2`) will need to manually update their `.github/workflows/pr-review.yml` to:

```yaml
uses: ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml@v2.0
```
**Note**: `v1.0` and `v2.0` are independent mutable tags. Updating one does not affect the other. Repositories pinned to `@v1.0` will continue to receive `v1.0.x` patches and will not automatically receive breaking changes from `v2.0`.

### Summary of Versioning Policy
| Tag	| Mutable?	| Receives Updates From	| Caller Update Required for... |
| :--- | :--- | :--- | :--- |
| `v1.0`	| Yes	| Latest `v1.0.x` patch	| Breaking changes only |
| `v1.0.0`, `v1.0.1` |	No (static)	| None (fixed commit)	| Never – pin to this for reproducibility |
| `v2.0`	| Yes	| Latest `v2.0.x` patch	| Yes – update `@v1.0` → `@v2.0` |
| `v2.0.0`	| No (static)	| None (fixed commit)	| Never – pin to this for reproducibility |


### Why This Approach?
- Patches are automatically applied – no need to touch every repository.
- Major versions are explicitly gated – callers must opt in.
- Static tags provide a fallback for reproducibility.
- No `@main` risk – we could simply use `@main`, but breaking changes in main can affect production repos and risky.

---
## How It Works (End-to-End)
1. PR created/updated → GitHub triggers the caller workflow.
2. Caller workflow invokes the reusable workflow from ollama_pr_review using the pinned tag (`@v1.0`).
3. Self-hosted runner on hypocentre picks up the job.
4. Workflow steps:
  - Fetches the PR diff.
  - Truncates it if too large (`MAX_BYTES=700000`).
  - Sends the diff to Ollama Cloud API (`deepseek-v4-flash:cloud`) with a structured prompt.
  - Receives a list of issues in a defined format.
  - Parses the response and extracts file names, line numbers, problems, and fixes.
  - Posts inline comments on the PR (with 6 lines of context).
5. Reviewers see the AI suggestions on the "Files changed" tab.

---
## Troubleshooting Common Issues
| Issue	| Likely Cause	| Fix |
| :--- | :--- | :--- |
| Runner shows "Idle" but jobs are queued	| Runner group access not granted to the repository	| Check organization Settings → Actions → Runner groups → Default → Repository access → enable public repos or add the repo |
| Empty review (❌ `No review content`)	| Diff too large; model runs out of tokens	| Reduce `MAX_BYTES` in the reusable workflow (e.g., to 300,000) |
| Jobs not being picked up	| Runner service not running	| `sudo ./svc.sh start` |
| Authentication errors	| Ollama API key expired or invalid	| Regenerate key and update GitHub secret |
| Stale duplicate runner	| Old runner still registered	| Remove it from the organization's Runners list (UI) |
| Reusable workflow not triggering	| Incorrect tag or syntax in caller workflow	| Check the uses line: `ucgmsim/ollama_pr_review/.github/workflows/pr-review.yml@v1.0` |
| Fork PRs not triggering	| The `if` condition skips forks	| This is intentional – forks don't have access to the OLLAMA_API_KEY secret |

## Tips for Team Members
- Reviews are suggestions, not final decisions – use your judgement.
- If a PR has many commits, only the latest review will complete (old ones are cancelled automatically).
- If the diff is too large, it is truncated and the first 700KB are reviewed. The current model deepseek-v4-flash has a 1M context size, but other models are typically more limited (150–255K), so they may need even smaller MAX_BYTES.
- The system uses Ollama Cloud – check usage here [https://ollama.com/settings].
- The reusable workflow is highly configurable. You can create another `.yml` using a different model to have multiple reviews.

## References
- Central reusable workflow (this repository): `ucgmsim/ollama_pr_review`
- Template repository: `ucgmsim/template`
- Caller workflow: `.github/workflows/pr-review.yml` in each repository.
