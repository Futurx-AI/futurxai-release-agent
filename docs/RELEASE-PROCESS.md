# Release Process

This document explains the automated 3-stage cherry-pick release pipeline.

---

## Pipeline Overview

```
develop ──PR merged──▶ Cherry-Pick Agent ──▶ release/v* branch
                              │
                              ▼
                        Deploy to QA ──▶ Automated Tests
                              │
                     ┌────────┴────────┐
                  ✅ Pass           ❌ Fail
                     │                 │
              qa-approved label   do-not-release label
                     │                 │
                     ▼                 ▼
              Deploy to Prod      Investigate & Fix
                     │
                     ▼
                main updated
```

---

## How It Works

### Stage 1 — Cherry-Pick Agent

**Trigger:** A PR is merged into `develop` with the `ready-for-release` label.

**What happens:**
1. The agent verifies the PR was actually merged (not just closed).
2. It confirms the merge commit exists on `develop`.
3. It checks that no release branch with the same name already exists.
4. It creates a new branch `release/v<YYYY-MM-DD>-<run_number>` from `main`.
5. It cherry-picks **only** the merge commit from the PR — it does NOT merge the entire `develop` branch.
6. If a cherry-pick conflict occurs, it aborts and posts a comment on the PR with the conflicting files.
7. On success, it pushes the release branch and comments on the PR.

### Stage 2 — QA Deploy Agent

**Trigger:** A `release/v*` branch is pushed.

**What happens:**
1. The release branch is deployed to the QA environment.
2. Automated tests run against QA.
3. If tests **pass**: the `qa-approved` label is added to the original PR.
4. If tests **fail**: the `do-not-release` label is added, and a failure comment is posted.

### Stage 3 — Production Deploy Agent

**Trigger:** The `qa-approved` label is added to a PR.

**What happens:**
1. The agent finds the most recent release branch.
2. It records the current `main` HEAD as a rollback point.
3. It merges the release branch into `main` with a no-fast-forward merge.
4. It deploys `main` to production.
5. On success, it posts a comment: "Deployed to production as release/v<date>. Commit: <hash>".
6. On deploy failure, it automatically rolls back `main` to the previous commit and force-pushes.

---

## How to Use

### For Developers

1. **Create your feature branch** from `develop`.
2. **Open a PR** targeting `develop`.
3. **When ready for release**, add the `ready-for-release` label to the PR.
4. **Merge the PR** into `develop` — the pipeline takes over from here.
5. **Monitor** the PR comments for status updates from the agents.

### Labels

| Label | Color | Meaning |
|-------|-------|---------|
| `ready-for-release` | 🟢 Green | PR should be cherry-picked into a release branch when merged |
| `qa-approved` | 🔵 Blue | QA tests passed — triggers production deploy |
| `do-not-release` | 🟡 Yellow | QA tests failed — do not deploy to production |

---

## Manual Rollback

If something goes wrong in production, you can trigger a manual rollback:

1. Go to **Actions** → **Manual Rollback** in the GitHub repository.
2. Click **Run workflow**.
3. Enter the **commit SHA** to roll back to (the previous good commit on `main`).
4. Optionally enter a **PR number** to post a notification comment.
5. Click **Run workflow**.

The rollback workflow will:
- Reset `main` to the specified commit.
- Force-push `main`.
- Redeploy production from the rolled-back state.
- Post a summary comment on the PR (if provided).

### Finding the rollback commit

To find the previous good commit, run:

```bash
git log --oneline main -10
```

Pick the commit SHA from before the problematic release merge.

---

## Customization

The deploy steps in each workflow contain placeholder commands. Replace them with your actual deployment commands:

- **QA Deploy** (`.github/workflows/deploy-qa.yml`): Replace the deploy and test steps.
- **Production Deploy** (`.github/workflows/deploy-prod.yml`): Replace the deploy step.
- **Rollback** (`.github/workflows/rollback.yml`): Replace the redeploy step.

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Cherry-pick conflict | Resolve manually, create the release branch by hand |
| QA tests failing | Check the Actions run log, fix the issue, create a new PR |
| Production deploy failed but auto-rollback succeeded | Investigate the deploy logs, then retry |
| Production deploy failed and auto-rollback also failed | Use the Manual Rollback workflow |
| Release branch already exists | A prior run created it; check if it was already deployed |
