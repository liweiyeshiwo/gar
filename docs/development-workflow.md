# Development Workflow

## 1. Route the task

Keep work local when requirements are changing, the task depends on local GUI/hardware/private services, or the final merge decision is involved.

Use Codex cloud when the task is bounded, pushed to GitHub, reproducible in an isolated environment, and verifiable with explicit commands.

## 2. Start locally

```powershell
git switch main
git pull --ff-only
git switch -c codex/<task-slug>
```

Write down the goal, non-goals, allowed scope, verification commands, and delivery format before implementation.

## 3. Create the handoff atom

```powershell
git status --short
git diff --check
git add -- <explicit-files>
git diff --cached --check
git commit -m "chore: checkpoint for cloud handoff"
git push -u origin codex/<task-slug>
git rev-parse HEAD
```

The printed SHA is the only accepted cloud handoff baseline. Uncommitted or unpushed changes are not part of the handoff.

## 4. Open a draft PR

Fill every section of `.github/pull_request_template.md`. Record the handoff SHA, current verification evidence, remaining work, risk, and rollback path.

## 5. Transfer branch ownership

State in the PR that the cloud agent now owns the branch. Do not write locally to that branch until the cloud task finishes or ownership is explicitly returned.

Start Codex cloud from the selected repository and branch/SHA, or use a non-review `@codex` PR instruction for PR-scoped follow-up work.

## 6. Review the cloud result

Review the summary and complete diff. Require GitHub Actions to pass. Use `@codex review` for semantic review, but do not treat it as a substitute for CI or human judgment.

## 7. Revalidate locally

```powershell
git fetch origin
git switch codex/<task-slug>
git pull --ff-only
git status --short
git diff --check origin/main...HEAD
git diff --stat origin/main...HEAD
```

Run every repository-specific validation command listed in `AGENTS.md`. Merge only after the working tree is clean, CI passes, discussions are resolved, and the diff matches the task.

## 8. Recover from failures

- If cloud installation or validation fails, stop expanding the diff, return branch ownership locally, and fix the repository contract or cloud environment before retrying.
- If cloud changes exceed the allowed scope, do not merge. Require an additive correction commit or restart from the recorded handoff SHA on a new branch.
- If local and cloud work conflict, stop one writer, preserve both commit histories, and resolve on a separate integration branch without force-push.
- If CI and local results disagree, compare runtime, lock files, environment variables, operating system, and command entry points; sediment the fix into versioned configuration.

## 9. Merge and sediment

Only the user merges. After merge:

```powershell
git switch main
git pull --ff-only
```

Move deterministic failures into tests or CI, repeated semantic constraints into `AGENTS.md`, and one-off context into the PR only.

## Cloud task prompt

```text
Base: the branch and commit SHA recorded in this draft PR
Goal: implement only the observable result stated in the PR
Non-goals: do not expand beyond the PR non-goals
Allowed scope: modify only the files and directories listed in the PR
Constraints: follow AGENTS.md; do not write to main; do not rewrite history
Verify: run every command listed in the PR verification section
Deliver: push only to this feature branch and summarize files, tests, and risks
```
