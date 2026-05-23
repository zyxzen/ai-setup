# /auto-triage Setup

Paste this file into Claude Code from inside a GitHub repo. Claude will:

1. Create six GitHub issue labels.
2. Install a `/auto-triage` slash command at `.claude/commands/auto-triage.md`.

## Prerequisites

- Current directory is a git repo with a GitHub remote.
- `gh` CLI is installed and authenticated (check with `gh auth status`).

If a prerequisite is missing, stop and tell the user — do not try to fix it.

---

## Action 1 — Create six GitHub labels

Create these labels on the repo. For each, run `gh label create <name> --description "<desc>" --color <hex>`; if it fails because the label already exists, skip it and continue.

| Name          | Color    | Description                                                  |
|---------------|----------|--------------------------------------------------------------|
| `bug`         | `d73a4a` | Incorrect behavior or broken functionality.                  |
| `improvement` | `a2eeef` | Working code that could be clearer, safer, or more correct.  |
| `performance` | `fbca04` | Slow, wasteful, or scalability-limited code paths.           |
| `structure`   | `bfd4f2` | Architecture / boundary / coupling problems.                 |
| `code-noise`  | `c5def5` | Stale comments, dead imports, formatting drift.              |
| `dead-code`   | `cfd3d7` | Unreachable or unused code that should be removed.           |

---

## Action 2 — Write `.claude/commands/auto-triage.md`

Create the directory `.claude/commands/` if it doesn't exist. Then write the file `.claude/commands/auto-triage.md` with this exact content:

````markdown
---
description: Scan codebase + open issues; report findings and file critical ones as GH issues.
---

You are running the `/auto-triage` slash command.

# Goal

Find critical problems in this codebase that are not yet tracked as GitHub issues, and file them. Surface advisory findings (code-noise, dead-code) in the terminal only.

# Prerequisites

- `gh` CLI authenticated for this repo (check with `gh auth status`).
- These six labels exist on the repo: `bug`, `improvement`, `performance`, `structure`, `code-noise`, `dead-code`. If any are missing, create them with `gh label create`.

If `gh` is not authenticated, stop and tell the user.

# Procedure

## 1. Fetch open issues

Run: `gh issue list --state open --limit 200 --json number,title,body,labels`

Hold the full list (titles + bodies) in context for dedup matching.

## 2. Scan the codebase

Look for findings in two tiers:

- **Critical** — security risk, broken behavior, data loss, or a blocking bug.
- **Advisory** — code-noise, dead-code, or anything below the critical bar.

Record each finding as: `file:line`, one-line description, severity (critical | advisory), category labels (one or more from the six).

## 3. Dedup critical findings

For each critical finding, semantically compare against the open issues fetched in step 1 (titles + bodies). If an existing issue clearly tracks the same problem, mark the finding as "tracked by #N" and do NOT file a new issue.

## 4. Print a terminal report

Group by category. Within each category, sort: critical-new first, then critical-tracked, then advisory. Each entry shows:

```
[severity] file:line — description (status)
```

Status is either `new` or `tracked by #N`.

## 5. File new GH issues

For every **critical-new** finding, create an issue:

```
gh issue create \
  --title "<short, action-oriented title>" \
  --body "<details, including file:line refs and reasoning>" \
  --label <cat1> [--label <cat2> ...]
```

No confirmation prompts. Advisory findings are never filed. If a single `gh issue create` fails, log the error and continue; summarize all failures at the end.

# Category labels

| Label         | Meaning                                                       |
|---------------|---------------------------------------------------------------|
| `bug`         | Incorrect behavior or broken functionality.                   |
| `improvement` | Working code that could be clearer, safer, or more correct.   |
| `performance` | Slow, wasteful, or scalability-limited code paths.            |
| `structure`   | Architecture / boundary / coupling problems.                  |
| `code-noise`  | Stale comments, dead imports, formatting drift, etc.          |
| `dead-code`   | Unreachable or unused code that should be removed.            |

`bug` and `performance` are typically critical when they affect correctness or production performance. `code-noise` and `dead-code` are always advisory. `improvement` and `structure` may be either, judged per finding. Multiple labels per issue are allowed.
````

---

## Done

Report what was done:
- Labels — list of `created` vs `already existed`
- `.claude/commands/auto-triage.md` — `created`

Then tell the user they can now invoke `/auto-triage` in Claude Code.
