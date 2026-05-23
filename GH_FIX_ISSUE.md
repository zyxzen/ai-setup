# /gh-fix-issue Setup

Paste this file into Claude Code from inside a GitHub repo. Claude will:

1. Install a `/gh-fix-issue` slash command at `.claude/commands/gh-fix-issue.md`.

After install, run `/gh-fix-issue 23` or `/gh-fix-issue #23` to fix a specific issue end-to-end: branch, commit, push, PR.

## Prerequisites

- Current directory is a git repo with a GitHub remote.
- `gh` CLI is installed and authenticated (check with `gh auth status`).

If a prerequisite is missing, stop and tell the user — do not try to fix it.

---

## Action 1 — Write `.claude/commands/gh-fix-issue.md`

Create the directory `.claude/commands/` if it doesn't exist. Then write the file `.claude/commands/gh-fix-issue.md` with this exact content:

````markdown
---
description: Fix a GitHub issue end-to-end — branch, implement, commit, push, PR.
argument-hint: <issue-number-with-or-without-#>
---

You are running the `/gh-fix-issue` slash command.

# Goal

Fix the GitHub issue the user passed in. End state: a branch is pushed and a PR is open against the default branch with `Closes #<N>` in the body.

# Input

The user invoked `/gh-fix-issue $ARGUMENTS`. Treat `$ARGUMENTS` as an issue reference — strip a leading `#` if present. If the result is not a positive integer, stop and tell the user.

# Prerequisites

- `gh` CLI authenticated for this repo (`gh auth status`).
- Working tree is clean (no uncommitted changes). If dirty, stop and tell the user.

If a prerequisite is missing, stop and tell the user — do not try to fix it.

# Procedure

## 1. Fetch the issue

Run: `gh issue view <N> --json number,title,body,labels,state,comments`

If state is `CLOSED`, stop and tell the user.

Print a short summary back: number, title, labels, one-line body excerpt.

## 2. Understand the problem

Read the full issue body and every comment. Scan the codebase for the relevant files (paths the issue references, or grep for symbols / strings mentioned).

If the fix is ambiguous — multiple valid approaches, missing context, the requested behavior conflicts with existing code, or the issue is vague — **STOP and ask the user**. Don't guess.

## 3. Create a branch

Get the default branch: `gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.

From the default branch, create and check out: `fix/issue-<N>-<slug>` (slug = 2-4 kebab-case words derived from the issue title).

## 4. Implement the fix

Make the minimal change that fixes the issue. Don't refactor unrelated code. Follow existing patterns in the touched files.

If the project has an obvious test / lint / type-check command (e.g., `npm test`, `pytest`, `cargo test`, `go test ./...`), run it after the change. If it fails, fix the regression before continuing. If no obvious command exists, skip.

## 5. Commit

Stage only the files you touched (no `git add -A` / `git add .`). Commit with:

```
<short imperative subject>

<1-3 sentence explanation of the fix>

Refs #<N>
```

Do NOT add Claude attribution — no `Co-Authored-By: Claude`, no `Generated with Claude Code` footer, nothing similar. Same for the PR body in the next step.

## 6. Push and open the PR

Push the branch: `git push -u origin <branch-name>`.

Open the PR. The issue MUST be referenced in BOTH the title and the body. The body MUST include the `Closes #<N>` keyword on its own line — that's what makes GitHub auto-close the issue when the PR is merged into the default branch:

```
gh pr create \
  --title "Fix #<N>: <commit subject>" \
  --body "Closes #<N>

<1-2 paragraph summary of the fix>

## Test plan
- [ ] <how to verify — e.g. 'reproduce the failing case from the issue'>
"
```

Print the PR URL.

# Done

Report:
- Issue: `#<N> <title>`
- Branch: `<branch-name>`
- PR: `<url>`
````

---

## Done

Report what was done:
- `.claude/commands/gh-fix-issue.md` — `created`

Then tell the user they can now invoke `/gh-fix-issue <number>` or `/gh-fix-issue #<number>` in Claude Code.
