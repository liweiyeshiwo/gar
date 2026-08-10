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
