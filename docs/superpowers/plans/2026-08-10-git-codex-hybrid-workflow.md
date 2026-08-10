# Git × Codex Hybrid Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and prove a local-first workflow in which local Codex hands reproducible tasks to Codex cloud through GitHub branches, commit SHAs, draft PRs, CI, review, and local revalidation.

**Architecture:** GitHub is the only shared state, a pushed commit SHA is the handoff atom, and a draft PR is the task interface. Local work owns requirements, environment-sensitive debugging, final verification, and merge decisions; Codex cloud owns bounded repository-reproducible work and PR-scoped follow-ups.

**Tech Stack:** Git 2.50.1+, Markdown, GitHub Actions YAML, Windows PowerShell 5.1 for local checks, Bash on `ubuntu-latest` for CI, `actions/checkout@v7.0.1`, and Codex cloud.

## Global Constraints

- The repository currently contains only the approved design and its primary-source research; do not invent an application language or business-code tree.
- Use `codex/<task-slug>` for task branches and keep each branch short-lived.
- GitHub is the only shared state; uncommitted or unpushed local changes are never part of a cloud handoff.
- Every cloud handoff is bound to a pushed commit SHA and a draft PR containing goal, non-goals, scope, verification, and delivery instructions.
- Only one writer may actively modify a task branch at a time.
- Agents never write directly to `main`, never force-push shared history, and never merge or publish.
- CI and local revalidation are hard gates; Codex review is an additional semantic review, not a replacement.
- Do not commit API keys, passwords, cookies, certificates, `.env` contents, or machine-specific secrets.
- Keep cloud-agent internet access off by default; enable only the minimum access required by an explicit task.
- Do not enable a required status check until the exact check has completed successfully at least once.
- No deletion is part of this plan.

---

## File Map

- `.gitattributes`: Normalize repository text to LF so Windows-local and Linux-cloud diffs stay stable.
- `.gitignore`: Exclude secrets, editor state, and OS metadata without hiding workflow artifacts.
- `README.md`: Orient humans and agents to the repository purpose and entry points.
- `AGENTS.md`: Define durable local/cloud commands, safety boundaries, completion rules, and semantic review rules.
- `docs/development-workflow.md`: Provide the operational local-to-cloud runbook.
- `.github/pull_request_template.md`: Require every PR to expose task scope, handoff SHAs, verification, risks, and rollback.
- `.github/workflows/ci.yml`: Provide one stable `quality` check for repository-contract validation.
- `docs/handoff-smoke-test.md`: Temporary-in-purpose but retained proof that the cloud handoff path has run end to end.

---

### Task 1: Establish the Repository Contract

**Files:**
- Create: `.gitattributes`
- Verify: `.gitignore` (established by approved preflight commit `38fc42c`)
- Create: `README.md`
- Create: `AGENTS.md`

**Interfaces:**
- Consumes: Approved design at `docs/superpowers/specs/2026-08-10-git-codex-hybrid-workflow-design.md`.
- Produces: Stable repository rules consumed by local Codex, Codex cloud, CI, and reviewers.

- [ ] **Step 1: Create cross-platform text and secret-exclusion rules**

Create `.gitattributes` with exactly:

```gitattributes
* text=auto
*.md text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
*.sh text eol=lf
*.ps1 text eol=lf
```

Verify that the `.gitignore` established by approved preflight commit `38fc42c` contains exactly:

```gitignore
# Local isolated worktrees
.worktrees/

# Secrets and local environment
.env
.env.*
!.env.example
*.key
*.pem
*.p12
*.pfx

# Editors and operating systems
.idea/
.vscode/
.DS_Store
Thumbs.db

# Local scratch output
.tmp/
tmp/
```

- [ ] **Step 2: Create the human entry point**

Create `README.md` with exactly:

```markdown
# gar

`gar` is a versioned software-development workflow for handing bounded work between a local developer, local Codex, GitHub, and Codex cloud.

## Operating model

- Local work owns requirements, architecture, environment-sensitive debugging, final verification, and merge decisions.
- Codex cloud owns bounded tasks that can be reproduced from a pushed branch or commit SHA.
- GitHub is the only shared state.
- Draft pull requests carry the task contract, evidence, discussion, and review.

## Start here

1. Read [`AGENTS.md`](AGENTS.md).
2. Follow [`docs/development-workflow.md`](docs/development-workflow.md).
3. Review the approved [design](docs/superpowers/specs/2026-08-10-git-codex-hybrid-workflow-design.md).
4. Consult the [primary-source research](docs/research/2026-08-10-git-codex-hybrid-workflow.md) when changing the workflow.

## Baseline validation

Before committing:

```powershell
git diff --check
git status --short
```

Before opening or updating a PR, inspect the staged or committed diff and confirm that it contains no secrets or machine-specific configuration.
```

- [ ] **Step 3: Create the shared agent contract**

Create `AGENTS.md` with exactly:

```markdown
# Repository Agent Contract

## Purpose

This repository defines and proves a local Codex × GitHub × Codex cloud development workflow. It does not yet contain an application technology stack.

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
```

- [ ] **Step 4: Verify the repository contract fails if a required file is missing, then passes after creation**

Run this PowerShell check from the repository root:

```powershell
$requiredFiles = @('.gitattributes', '.gitignore', 'README.md', 'AGENTS.md')
$invalidFiles = $requiredFiles | Where-Object {
    -not (Test-Path -LiteralPath $_) -or (Get-Item -LiteralPath $_).Length -eq 0
}
if ($invalidFiles) { throw "Missing or empty required files: $($invalidFiles -join ', ')" }
git diff --check
```

Expected: exit code `0`, no missing-file exception, and no whitespace errors.

- [ ] **Step 5: Commit the repository contract**

```powershell
git add -- .gitattributes README.md AGENTS.md
git diff --cached --check
git commit -m "chore: establish repository workflow contract"
```

Expected: `.gitignore` remains the final contract file established by approved preflight commit `38fc42c` and is verified only in Task 1. One Task 1 commit contains only `.gitattributes`, `README.md`, and `AGENTS.md`.

---

### Task 2: Add the Operational Runbook and PR Contract

**Files:**
- Create: `docs/development-workflow.md`
- Create: `.github/pull_request_template.md`

**Interfaces:**
- Consumes: Git and safety rules from `AGENTS.md`.
- Produces: A concrete handoff procedure and the exact PR fields required by CI, Codex cloud, and reviewers.

- [ ] **Step 1: Create the operational runbook**

Create `docs/development-workflow.md` with exactly:

```markdown
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
```

- [ ] **Step 2: Create the PR contract**

Create `.github/pull_request_template.md` with exactly:

```markdown
## Goal

Describe the observable result.

## Non-goals

List work intentionally excluded from this PR.

## Allowed scope

List the files or directories that may change.

## Ownership

- Current writer: local user / local Codex / Codex cloud
- Ownership transferred at: branch and commit SHA

## Handoff

- Source branch:
- Local handoff SHA:
- Cloud final SHA:

## Verification

List each command and its result.

## Risks and uncovered cases

Describe remaining uncertainty.

## Rollback

Describe how to revert or disable the change.

## Checklist

- [ ] Diff matches the goal and allowed scope
- [ ] No secrets or machine-specific configuration
- [ ] Local verification passed
- [ ] GitHub Actions passed
- [ ] Review conversations resolved
- [ ] Cloud output revalidated locally
- [ ] User retains merge and release authority
```

- [ ] **Step 3: Verify links, required headings, and whitespace**

Run:

```powershell
$workflow = Get-Content -Raw -Encoding utf8 docs/development-workflow.md
$template = Get-Content -Raw -Encoding utf8 .github/pull_request_template.md
@('# Development Workflow', '## Cloud task prompt') | ForEach-Object {
    if (-not $workflow.Contains($_)) { throw "Missing workflow heading: $_" }
}
@('## Goal', '## Handoff', '## Verification', '## Rollback') | ForEach-Object {
    if (-not $template.Contains($_)) { throw "Missing PR heading: $_" }
}
git diff --check
```

Expected: exit code `0` and no missing-heading or whitespace errors.

- [ ] **Step 4: Commit the runbook and PR contract**

```powershell
git add -- docs/development-workflow.md .github/pull_request_template.md
git diff --cached --check
git commit -m "docs: add local to cloud handoff runbook"
```

Expected: one commit containing only the runbook and PR template.

---

### Task 3: Add the Stable GitHub Actions Quality Gate

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: Required contract files from Tasks 1 and 2.
- Produces: One stable GitHub check named `quality`, plus an intentionally failing signal for the later handoff rehearsal.

Bootstrap note: the user approved a one-time controller push of commit `38fc42c` to create `origin/main`. This is the only direct `main` bootstrap; Task 1-3 implementation remains on `codex/hybrid-workflow` and returns through a PR.

- [ ] **Step 1: Create the CI workflow**

Create `.github/workflows/ci.yml` with exactly:

```yaml
name: quality

on:
  push:
    branches:
      - main
  pull_request:

permissions:
  contents: read

jobs:
  quality:
    name: quality
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v7.0.1

      - name: Validate repository contract
        shell: bash
        run: |
          set -euo pipefail

          required_files=(
            .gitattributes
            .gitignore
            README.md
            AGENTS.md
            docs/development-workflow.md
            .github/pull_request_template.md
            .github/workflows/ci.yml
          )

          for file in "${required_files[@]}"; do
            if [[ ! -s "$file" ]]; then
              echo "::error file=$file::required file is missing or empty"
              exit 1
            fi
          done

          if git grep -nI -E '[[:blank:]]+$' -- "${required_files[@]}"; then
            echo "::error::contract files contain trailing whitespace"
            exit 1
          fi

          if git grep -nI -E 'TBD|TODO|待定|未确定|待补' -- "${required_files[@]}"; then
            echo "::error::contract files contain unresolved placeholders"
            exit 1
          fi

          if [[ -f docs/handoff-smoke-test.md ]]; then
            grep -Fxq 'Status: cloud-complete' docs/handoff-smoke-test.md || {
              echo "::error file=docs/handoff-smoke-test.md::cloud handoff is not complete"
              exit 1
            }
          fi
```

- [ ] **Step 2: Verify the workflow structure locally**

Run:

```powershell
$ci = Get-Content -Raw -Encoding utf8 .github/workflows/ci.yml
@('name: quality', 'actions/checkout@v7.0.1', 'name: quality', 'docs/handoff-smoke-test.md') | ForEach-Object {
    if (-not $ci.Contains($_)) { throw "Missing CI contract text: $_" }
}
git diff --check
```

Expected: exit code `0`; the file contains the stable check name, immutable checkout release, contract validation, and handoff-smoke-test gate.

- [ ] **Step 3: Commit the CI gate**

```powershell
git add -- .github/workflows/ci.yml
git diff --cached --check
git commit -m "ci: add repository quality gate"
```

Expected: one commit containing only `.github/workflows/ci.yml`.

- [ ] **Step 4: Perform the pre-push verification**

Run:

```powershell
git status --short
git diff --check 38fc42c..HEAD
git log --oneline 38fc42c..HEAD
```

Expected: a clean working tree and focused Task 1-3 commits after the approved bootstrap base `38fc42c`.

- [ ] **Step 5: Push the reviewed feature branch**

This step and Step 6 are controller-owned remote gates performed only after the per-task local review approves the CI commit. The implementer stops after Step 4.

Run:

```powershell
git push -u origin codex/hybrid-workflow
```

Expected: `origin/codex/hybrid-workflow` is created; `origin/main` remains at the approved bootstrap commit until the user merges the PR.

- [ ] **Step 6: Open the workflow PR and verify the first real `quality` check**

Open a draft PR from `codex/hybrid-workflow` to `main`, fill the repository PR template, and verify the `quality` workflow completed successfully for the PR head. Record the exact visible check name `quality`.

Expected: one successful `quality` check attached to the PR head. Stop here if the check has not succeeded; do not create a required check by guessed name.

---

### Task 4: Configure GitHub and Codex Cloud Boundaries

**Files:**
- No repository file changes.

**Interfaces:**
- Consumes: The bootstrap `origin/main`, the pushed `codex/hybrid-workflow` branch, its draft PR, and a proven `quality` check from Task 3.
- Produces: Protected `main`, a least-privilege Codex cloud repository connection, and a cloud environment that reads the repository contract.

- [ ] **Step 1: Protect `main` only after the successful CI run exists**

In the GitHub repository settings, create a branch ruleset targeting `main` with these exact requirements:

```text
Require a pull request before merging: enabled
Required approvals: 0 for the current single-user repository
Require status checks to pass: enabled
Required status check: quality
Require conversation resolution: enabled
Block force pushes: enabled
Block deletions: enabled
```

Expected: direct force pushes and branch deletion are blocked, while the single owner can merge a PR after the deterministic gates pass.

- [ ] **Step 2: Connect only this repository to Codex cloud**

In Codex cloud settings, authorize `liweiyeshiwo/gar` and do not grant organization-wide or all-repository access.

Expected: `gar` appears as a selectable Codex cloud repository and unrelated repositories remain unselected.

- [ ] **Step 3: Create the minimal cloud environment**

Configure the environment with these values:

```text
Repository: liweiyeshiwo/gar
Base branch: main
Setup script: empty
Maintenance script: empty
Environment variables: none
Secrets: none
Agent internet access: off
```

Expected: a cloud task can check out `main`, read `AGENTS.md`, and run Git commands without receiving secrets or unnecessary network access.

- [ ] **Step 4: Verify the cloud environment without writing**

Run a read-only Codex cloud task with this exact prompt:

```text
Read AGENTS.md and docs/development-workflow.md. Do not modify files. Report the current branch, current commit SHA, required pre-commit commands, branch ownership rule, and who may merge.
```

Expected: the response identifies `git diff --check`, `git status --short`, the one-writer rule, and the user as the only merge decision-maker; the task returns no diff.

- [ ] **Step 5: Let the user merge the reviewed workflow baseline**

After branch protection is active, the PR `quality` check passes, review conversations are resolved, and local revalidation is clean, the user converts the draft PR to ready and merges it.

Expected: `origin/main` contains the repository contract, runbook, PR template, and CI workflow; the push-triggered `quality` run on `main` also succeeds.

---

### Task 5: Prove the Local-to-Cloud Handoff End to End

**Files:**
- Create locally: `docs/handoff-smoke-test.md`
- Modify in Codex cloud: `docs/handoff-smoke-test.md`

**Interfaces:**
- Consumes: Protected `main`, proven `quality` check, Codex cloud connection, `AGENTS.md`, and PR template.
- Produces: A draft PR whose CI fails before cloud work, passes after cloud work, and is revalidated locally before user merge.

- [ ] **Step 1: Create the rehearsal branch**

Run:

```powershell
git fetch origin
git switch -c codex/handoff-smoke-test origin/main
```

Expected: the active branch is `codex/handoff-smoke-test` and starts from the protected `main` baseline.

- [ ] **Step 2: Create the intentionally incomplete handoff artifact**

Create `docs/handoff-smoke-test.md` with exactly:

```markdown
# Cloud Handoff Smoke Test

Status: pending-cloud

## Goal

Prove that Codex cloud can read the repository contract, make one scoped change, run the declared checks, push to the feature branch, and return evidence through the draft PR.

## Allowed cloud change

Change `Status: pending-cloud` to `Status: cloud-complete` and append a `## Cloud evidence` section containing the commit SHA and the commands run. Do not modify any other file.
```

- [ ] **Step 3: Verify the branch is intentionally red before handoff**

Run:

```powershell
git diff --check
$statusLine = Select-String -Path docs/handoff-smoke-test.md -Pattern '^Status: pending-cloud$'
if (-not $statusLine) { throw 'Expected pending-cloud status before handoff' }
```

Expected: exit code `0` and exactly one pending status line.

- [ ] **Step 4: Commit and push the handoff atom**

Run:

```powershell
git add -- docs/handoff-smoke-test.md
git diff --cached --check
git commit -m "test: start cloud handoff smoke test"
git push -u origin codex/handoff-smoke-test
$handoffSha = git rev-parse HEAD
$handoffSha
```

Expected: the branch exists on GitHub and `$handoffSha` prints the immutable local handoff baseline.

- [ ] **Step 5: Open a draft PR and record the concrete SHA**

Use the GitHub web interface to open a draft PR from `codex/handoff-smoke-test` to `main`. Fill the repository PR template with these facts:

```text
Goal: Complete the cloud handoff smoke test.
Non-goals: No application code, workflow redesign, dependency changes, or edits outside docs/handoff-smoke-test.md.
Allowed scope: docs/handoff-smoke-test.md only.
Current writer: Codex cloud after task launch.
Ownership transferred at: codex/handoff-smoke-test at the SHA printed in Step 4.
Local handoff SHA: the SHA printed in Step 4.
Verification: git diff --check passed; pending-cloud status confirmed.
Risk: Documentation-only rehearsal.
Rollback: Revert the two smoke-test commits through a PR.
```

Expected: the draft PR shows the `quality` check failing specifically because the smoke-test status is not yet `cloud-complete`.

- [ ] **Step 6: Hand ownership to Codex cloud with an exact task**

Launch Codex cloud on the draft PR branch or issue a non-review `@codex` PR instruction with this exact prompt:

```text
Base: codex/handoff-smoke-test at the Local handoff SHA recorded in this draft PR
Goal: complete the repository's cloud handoff smoke test
Non-goals: do not redesign the workflow and do not modify any other file
Allowed scope: docs/handoff-smoke-test.md only
Constraints: follow AGENTS.md; do not write to main; do not rewrite history
Verify: run git diff --check and confirm the file contains exactly one line equal to Status: cloud-complete
Deliver: commit with message "test: complete cloud handoff smoke test", push only to codex/handoff-smoke-test, and summarize the final SHA and verification results
```

Expected cloud diff in `docs/handoff-smoke-test.md`:

```markdown
Status: cloud-complete

## Cloud evidence

- Commit SHA: the final cloud commit reported in the PR
- `git diff --check`: passed
- Status-line check: passed
```

- [ ] **Step 7: Verify cloud delivery and semantic review**

In GitHub:

1. Confirm only `docs/handoff-smoke-test.md` changed after the handoff SHA.
2. Confirm the new commit message is `test: complete cloud handoff smoke test`.
3. Confirm `quality` now passes.
4. Request `@codex review`.
5. Resolve every high-severity finding or explain why it is not applicable.

Expected: the draft PR is green, scoped to one file, and contains both local and cloud commit SHAs plus verification evidence.

- [ ] **Step 8: Return ownership and revalidate locally**

Run:

```powershell
git fetch origin
git switch codex/handoff-smoke-test
git pull --ff-only
git status --short
git diff --check origin/main...HEAD
git diff --name-only origin/main...HEAD
$completed = Select-String -Path docs/handoff-smoke-test.md -Pattern '^Status: cloud-complete$'
if (-not $completed) { throw 'Cloud completion status is missing' }
```

Expected: clean working tree; no whitespace errors; only `docs/handoff-smoke-test.md` differs from `origin/main`; completion status is present.

- [ ] **Step 9: Let the user merge, then verify `main`**

After the user converts the draft PR to ready and merges it, run:

```powershell
git fetch origin
git status --short
$mergedSmokeTest = git show origin/main:docs/handoff-smoke-test.md
$mergedSmokeTest | Select-String -Pattern '^Status: cloud-complete$'
git log -1 --oneline origin/main
```

Expected: the worktree is clean, `origin/main` contains the completed smoke-test artifact, and its latest history contains the merged PR.

---

### Task 6: Final Workflow Audit

**Files:**
- Modify only if evidence requires it: `AGENTS.md`
- Modify only if evidence requires it: `docs/development-workflow.md`

**Interfaces:**
- Consumes: The merged handoff rehearsal and all verification output.
- Produces: A verified workflow with no speculative sediment.

- [ ] **Step 1: Audit the implementation against the approved design**

Confirm each statement with repository or GitHub evidence:

```text
GitHub is the only shared state.
Every cloud handoff is bound to a pushed SHA.
The draft PR carries scope and verification.
Only one writer owns the branch at a time.
The cloud agent cannot write directly to main.
The quality check is required and passing.
Codex review did not replace CI or local revalidation.
The user made the merge decision.
```

Expected: every statement is supported by a file, commit, PR field, check, or settings screen.

- [ ] **Step 2: Sediment only an observed reusable failure**

If the rehearsal exposed a deterministic reproducibility failure, add the smallest exact correction to `AGENTS.md`, `docs/development-workflow.md`, or CI. If no reusable failure occurred, make no repository change and record that fact in the final report only.

For any correction, run:

```powershell
git diff --check
git diff --stat
```

Expected: no speculative rules; any change directly corresponds to observed evidence.

- [ ] **Step 3: Commit an evidence-driven correction only when one exists**

If Step 2 changed a file:

```powershell
git add -- AGENTS.md docs/development-workflow.md .github/workflows/ci.yml
git diff --cached --check
git commit -m "docs: incorporate cloud handoff findings"
```

If Step 2 made no change, skip the commit and preserve the clean working tree.

- [ ] **Step 4: Run the final local verification**

Run:

```powershell
git fetch origin
git status --short --branch
git diff --check
git log --oneline --decorate -8 origin/main
```

Expected: a clean worktree, no whitespace errors, and `origin/main` history showing the baseline, workflow, CI, and handoff rehearsal.

---

## Completion Evidence

Implementation is complete only when all of the following are observable:

- `README.md`, `AGENTS.md`, the runbook, PR template, and CI workflow exist and are non-empty.
- `origin/main` exists and has a successful `quality` check.
- `main` protection requires PR, `quality`, and conversation resolution while blocking force-push and deletion.
- Codex cloud can read the repository without creating a diff.
- The smoke-test PR fails before cloud completion and passes after it.
- The cloud change is limited to `docs/handoff-smoke-test.md`.
- The final cloud commit is pulled and revalidated locally.
- The user, not an agent, decides and performs the merge.
- The final local `main` is clean and contains `Status: cloud-complete`.

## Primary References

- [OpenAI: Codex cloud](https://learn.chatgpt.com/docs/cloud)
- [OpenAI: Cloud environments](https://learn.chatgpt.com/docs/environments/cloud-environment)
- [OpenAI: AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)
- [OpenAI: GitHub integration and code review](https://learn.chatgpt.com/docs/third-party/github)
- [GitHub: GitHub flow](https://docs.github.com/en/get-started/using-github/github-flow)
- [GitHub: Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [GitHub: actions/checkout v7.0.1](https://github.com/actions/checkout/releases/tag/v7.0.1)
