# Repository Agent Contract

## Purpose

This repository defines and proves a local Codex 脳 GitHub 脳 Codex cloud development workflow. It does not yet contain an application technology stack.

## Read order

1. Read `README.md`.
2. Read `docs/development-workflow.md` before changing workflow files.
3. Read the approved design in `docs/superpowers/specs/` when a decision is unclear.

## Git rules

- Start unrelated work on a short `codex/<task-slug>` branch.
- Never write directly to `main`.
- Never force-push shared history.
- Never delete files, directories, branches, or remote data as part of routine work.
- Before a cloud handoff, commit and push the branch and record `git rev-parse HEAD` in the draft PR.
- One writer owns a task branch at a time. Transfer ownership only after commit and push.

## Validation

Run these commands before every commit:

```powershell
git diff --check
git status --short
```

Before every PR update, inspect the full branch diff:

```powershell
git diff --check origin/main...HEAD
git diff --stat origin/main...HEAD
```

When `.github/workflows/ci.yml` exists, the PR is not mergeable until its `quality` job succeeds.

## Definition of Done

- The diff matches the stated task and allowed scope.
- Required local commands pass.
- GitHub Actions passes.
- PR conversations are resolved.
- High-severity review findings are addressed.
- The local developer revalidates the cloud result.
- Only the user decides whether to merge or publish.

## Security

- Never commit secrets, `.env` contents, credentials, cookies, certificates, or machine-specific access tokens.
- Keep cloud-agent internet access off unless the task explicitly requires a limited allowlist.
- Agents do not merge, publish, deploy, or change production data.

## Code Review Rules

- Reject a handoff whose PR does not identify both the handoff commit SHA and the verification evidence.
- Reject concurrent local and cloud writes to the same branch unless ownership transfer is explicit.
- Reject changes that bypass CI, local revalidation, branch protection, or the user's merge authority.
