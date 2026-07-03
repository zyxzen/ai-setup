---
description: Take a Jira ticket from key to PR — fetch, clarify, plan, implement with stack agents, review, address feedback, open PR. Fully automated except two human gates.
argument-hint: <JIRA-TICKET-KEY e.g. SCRUM-1118>
---

You are running the `/do-ticket` slash command.

# Goal

Take the Jira ticket identified by `$ARGUMENTS` from key to an open pull request,
fully automated. There are exactly **two** points where you stop for a human:

1. The ticket has genuine open questions (clarity gate, step 2).
2. Blocking review issues remain after 3 review rounds (step 6).

Everywhere else: no interaction. Don't ask the user to confirm steps — just do them.

# Input

`$ARGUMENTS` is a Jira issue key like `SCRUM-1118`. Uppercase it and strip
surrounding whitespace. If it doesn't look like `<PROJECT>-<NUMBER>`, stop and tell
the user. Below, `<ID>` = that key, `<id>` = its lowercase form.

# Prerequisites

Check these first. If any is missing, **stop and tell the user — do not try to fix it.**

- Current directory is a git repository.
- An Atlassian MCP server is configured for this project — i.e. tools named
  `mcp__atlassian-<slug>__*` are available (set up via `/setup-project-accounts`).
  Use whichever Atlassian server is present; **never hardcode a slug**.
- The stack agents are installed for this repo: `rails-engineer`/`rails-reviewer`
  (Rails), `react-engineer`/`react-reviewer`/`ux-reviewer` (React), and
  `software-engineer`.
- `gh` is authenticated (`gh auth status`) — needed for the PR.
- Working tree is clean. If dirty, stop and tell the user.

---

# Procedure

## 1. Fetch the ticket (Atlassian MCP)

Find the available Atlassian tools by their `mcp__atlassian-<slug>__` prefix and use
that slug for all Atlassian calls.

1. Call `…getAccessibleAtlassianResources` to obtain the `cloudId`. If more than one
   resource is returned, pick the one whose project matches `<ID>`'s project prefix.
2. Call `…getJiraIssue` with that `cloudId` and `issueIdOrKey: <ID>`, requesting the
   summary, description, status, issue type, labels, and acceptance-criteria fields.
   Also fetch the issue's comments (use the dedicated comments tool if one exists,
   otherwise the rendered/comment fields on the issue).

Write `.tickets/<ID>/task.md` (create dirs as needed) with:

```markdown
# <ID>: <summary>

- **Status:** <status>   **Type:** <issue type>   **Labels:** <labels>
- **Link:** <browse URL for the issue>

## Description
<full description>

## Acceptance criteria
<criteria, as a checklist if the ticket lists them>

## Comments
<each comment: author — text>
```

Print a one-line summary back: `<ID> — <summary> [<status>]`.

## 2. Clarity gate → `.tickets/<ID>/task-questions.md`

Read `task.md` with a skeptical eye. Decide whether the ticket is **clear enough to
implement without guessing**: are the acceptance criteria unambiguous, is the
expected behavior defined, do referenced systems/endpoints/designs exist in the repo?

**Always create `.tickets/<ID>/task-questions.md`.**

- **If there are genuine open questions** (ambiguity that would change the
  implementation, missing decisions, conflicting requirements), write:

  ```markdown
  # Open questions for <ID>

  STATUS: BLOCKED — answer each question inline below, then re-run `/do-ticket <ID>`.

  1. <question>
     **Answer:**
  2. <question>
     **Answer:**
  ```

  Then **STOP** and tell the user the run is paused on open questions in that file.
  Do not proceed.

- **On a re-run**, if `task-questions.md` exists with the answers filled in, treat
  those answers as part of the spec and re-evaluate clarity. Proceed once clear.

- **If the ticket is clear**, write:

  ```markdown
  # Open questions for <ID>

  STATUS: CLEAR — no open questions. Proceeding automatically.
  ```

  and continue with **no further interaction** through to the PR.

## 3. Plan & branch

Detect the stack(s) present in the repo:

- **Rails** if `Gemfile`/`config/routes.rb`/`app/models` exist.
- **React** if `package.json` lists `react` (or `next`).
- If **both**, this is a full-stack ticket — both pipelines apply.
- If **neither**, stop and tell the user (no matching engineer agent).

Write `.tickets/<ID>/plan.md`: the approach, the files/areas to change (split into
backend / frontend when full-stack), and how each acceptance criterion will be met.

Get the default branch:
`gh repo view --json defaultBranchRef -q .defaultBranchRef.name`.
From it, create and check out a branch named `<ID>`, e.g. `SCRUM-1118`.

## 4. Implement (engineer agents)

Dispatch the engineer agent(s) for the detected stack, giving each the ticket
(`task.md`), the answers, and the plan:

- Rails work → **`rails-engineer`**.
- React work → **`react-engineer`**.
- Full-stack → both. Do the backend (`rails-engineer`) first when the frontend
  depends on it; run them in parallel only when the slices are independent.

Make the minimal change that satisfies the acceptance criteria; follow existing
patterns; don't refactor unrelated code. After the change, run the project's obvious
test/lint/type-check command (e.g. `bundle exec rspec`, `npm test`, `npm run lint`)
and fix any regression before moving on.

## 5. Review (fully automated)

Dispatch the reviewers for the detected stack(s) **in parallel**, giving each the
diff and the ticket. Collect their findings. Reviewer set:

- Rails → `software-engineer`, `rails-reviewer`.
- React → `software-engineer`, `react-reviewer`, `ux-reviewer`.
- Full-stack → the union of both, with `software-engineer` run **once**.

Treat each finding's severity as the reviewer states it. "Blocking" = correctness,
security, broken acceptance criteria, or anything a reviewer marks must-fix.

## 6. Address feedback → re-review (max 3 rounds)

Loop, up to **3 rounds total**:

1. Apply fixes for every blocking finding (use the engineer agent for non-trivial
   changes). Re-run tests/lint.
2. Re-review the changed areas with the same reviewer set.
3. If no blocking findings remain → exit the loop, go to step 7.

If blocking findings still remain after the 3rd round, **STOP**: write them to
`.tickets/<ID>/review-blockers.md` (each with file, severity, and what's needed) and
tell the user. Do **not** open a PR.

## 7. Commit, push, open PR

Only reached when no blocking issues remain.

1. Stage only the files you touched (no `git add -A`/`.`). Commit with an imperative
   subject, a short body, and `Refs <ID>`. **No Claude attribution** anywhere.
2. Push: `git push -u origin <ID>`.
3. Read `PR_FORMAT.md` — prefer one in the target repo root; if absent, use the
   template that ships in this `ai-setup` repo. Build the PR title and body from it,
   filling the ticket link from `task.md` and the test plan from the acceptance
   criteria. The ticket key MUST appear in both title and body.
4. `gh pr create --title "<ID>: <subject>" --body "<rendered body>"`.
5. Print the PR URL.

# Done

Report:

- Ticket: `<ID> — <summary>`
- Branch: `<ID>`
- Stack(s): `<rails | react | both>`
- Review: `<rounds used>` round(s), blockers resolved
- PR: `<url>`

If you stopped at a gate instead, report which gate and the path to the file the user
needs to act on (`task-questions.md` or `review-blockers.md`).
