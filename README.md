# futurxai-release-agent

Automated cherry-pick release pipeline for FuturxAI.

## Overview

This repository implements a 3-stage automated release pipeline:

1. **Cherry-Pick** — When a PR with `ready-for-release` label is merged into `develop`, the agent cherry-picks the merge commit onto a new `release/v<date>-<run>` branch.
2. **QA Deploy** — The release branch is automatically deployed to QA. If tests pass, `qa-approved` is added to the PR.
3. **Production Deploy** — When `qa-approved` is added, the release branch is merged into `main` and deployed to production.

A manual **Rollback** workflow is also available from the GitHub Actions UI.

See [docs/RELEASE-PROCESS.md](docs/RELEASE-PROCESS.md) for the full guide.
