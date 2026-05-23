# /gh-close-duplicates Setup

Paste this file into Claude Code from inside a GitHub repo. Claude will:

1. Install a `/gh-close-duplicates` slash command at `.claude/commands/gh-close-duplicates.md`.

After install, run `/gh-close-duplicates` to find clusters of duplicate open issues, pick the oldest in each cluster as canonical, and close the rest after a single confirmation.

## Prerequisites

- Current directory is a git repo with a GitHub remote.
- `gh` CLI is installed and authenticated (check with `gh auth status`).

If a prerequisite is missing, stop and tell the user — do not try to fix it.

---

## Action 1 — Write `.claude/commands/gh-close-duplicates.md`

Create the directory `.claude/commands/` if it doesn't exist. Then write the file `.claude/commands/gh-close-duplicates.md` with this exact content:

````markdown
---
description: Find duplicate open issues, pick the oldest as canonical, close the rest after confirmation.
---

You are running the `/gh-close-duplicates` slash command.

# Goal

Cluster open issues by semantic similarity. In each cluster of 2+, keep the oldest as canonical and close the rest with a `Duplicate of #N` comment — after a single confirmation from the user.

# Prerequisites

- `gh` CLI authenticated for this repo (`gh auth status`).

If not authenticated, stop and tell the user.

# Procedure

## 1. Fetch open issues

Run: `gh issue list --state open --limit 200 --json number,title,body,createdAt,labels`

Hold the full list (titles + bodies) in context.

## 2. Cluster by semantic similarity

Group issues that describe the same underlying problem. Use full titles + bodies — not just titles. A cluster needs at least 2 issues to be relevant.

Be conservative: if you're not confident two issues are the same problem, treat them as separate. False positives (closing a real issue) are worse than false negatives (missing a dupe).

For each cluster of 2+, pick the **oldest** issue (earliest `createdAt`, tiebreak by lowest number) as canonical. The rest are duplicates of it.

If there are no clusters, tell the user "no duplicates found" and stop.

## 3. Print the plan

For each cluster, print:

```
Canonical: #<N> <title>
Duplicates to close:
  - #<A> <title>  — <one-line reason it's a dupe>
  - #<B> <title>  — <one-line reason it's a dupe>
```

At the bottom: `Will close <total> issues as duplicates.`

## 4. Confirm

Ask the user: `Close these <total> issues? (yes/no)`

If the response is anything other than `yes` or `y`, stop without closing anything.

## 5. Close duplicates

For each duplicate, run:

```
gh issue close <dupe-N> \
  --reason "not planned" \
  --comment "Duplicate of #<canonical-N>."
```

If a single close fails, log the error and continue. Summarize all failures at the end.

# Done

Report:
- Closed: list of `#<N> → duplicate of #<canonical-N>`
- Failures (if any)
````

---

## Done

Report what was done:
- `.claude/commands/gh-close-duplicates.md` — `created`

Then tell the user they can now invoke `/gh-close-duplicates` in Claude Code.
