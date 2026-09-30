# Contributing to Splitbook (app)

Thanks for working on Splitbook. This guide covers how work moves from an issue to `main`.

## Where work comes from

- All work is tracked as issues on the **Splitbook v1** project board.
- Pick up issues assigned to you, and move the card as you go: **Todo → In progress → In review**.
- **Done** is set by the reviewer, not by the person doing the work.
- If something isn't covered by an issue, ask for one to be created before starting.

## Workflow

1. Make sure your local `main` is up to date: `git checkout main && git pull`.
2. Create a branch for the issue (see naming below).
3. Commit your work in small, clear steps.
4. Push the branch and open a pull request into `main`.
5. Link the issue in the PR description with `Closes #<issue-number>`.
6. Move the card to **In review** and wait for approval.
7. Address review comments with new commits on the same branch. A new push dismisses the previous approval, so the reviewer will approve again.
8. Once approved and all conversations are resolved, the PR is **squash-merged** into `main`.

`main` is protected: direct pushes, force-pushes and self-approval are blocked.

## Branch naming

| Type | Pattern | Example |
|---|---|---|
| New feature | `feature/<short-name>` | `feature/workout-tabs` |
| Bug fix | `fix/<short-name>` | `fix/split-rounding` |
| Documentation | `docs/<short-name>` | `docs/readme-setup` |
| Setup or tooling | `chore/<short-name>` | `chore/eslint` |

Use lowercase and hyphens, and keep names short.

## Commits and pull requests

- Write commit messages in the imperative, describing what the commit does: "Add workout tab list", not "Added stuff".
- One issue per PR. If a PR grows beyond its issue, split it.
- Fill in every section of the PR template.
- For UI changes, include screenshots or a short screen recording.
- If a change alters behavior described in `docs/SPEC.md`, update the spec in the same PR.

## Research issues

Research issues are answered with a comment on the issue, not a PR. Use this format for each checklist item:

```
**Item:** <checklist item>
**Answer:** <what the official docs say, in your own words>
**Source:** <link to the official page>
**Checked on:** <date>
**Impact on spec:** None / Needs change — <what should change>
```

Only official documentation counts as a source. If something can't be confirmed, write "Could not confirm" and explain what you found.

## Secrets and data (this repo is public)

- **Never commit secrets:** API keys, client secrets, tokens, passwords, keystores or signing files.
- Local values go in `.env`, which is ignored by git. `.env.example` lists variable names only, with no real values.
- OAuth client secrets and tokens belong in the backend, never in the app.
- **Never commit real health data**, including your own workouts as test files. Use made-up sample data.
- If you ever commit a secret, **tell the maintainer immediately.** Deleting the file isn't enough, because git history keeps it; the secret must be rotated.

GitHub secret scanning with push protection is enabled and will block pushes that look like they contain secrets. Don't bypass it; ask instead.

## Code style

Linting and formatting tools will be set up in a later issue. Until then, keep code consistent with the surrounding files and use TypeScript for all app code.

## Questions

Ask in the issue you're working on, so the answer stays with the work.