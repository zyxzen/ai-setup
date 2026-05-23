# /auto-triage CI Setup

Paste this file into Claude Code from inside a GitHub repo. Claude will:

1. Ask which branch the workflow should run on (default: `main`).
2. Ask the schedule (default: daily at 09:00 UTC).
3. Write a GitHub Actions workflow at `.github/workflows/auto-triage.yml` that runs `/auto-triage` on that schedule.

## Prerequisites

- Current directory is a git repo with a GitHub remote.
- `.claude/commands/auto-triage.md` already exists in the repo. If it doesn't, stop and tell the user to paste `AUTO_TRIAGE.md` first.
- `gh` CLI is installed and authenticated (check with `gh auth status`).

If a prerequisite is missing, stop and tell the user — do not try to fix it.

---

## Action 1 — Ask for branch and schedule

Ask the user, one question at a time:

1. **Branch:** "Which branch should the auto-triage workflow check out and run against? (default: `main`)"
2. **Schedule:** "How often should it run? Pick one — `daily` (default, 09:00 UTC), `weekly` (Mondays 09:00 UTC), `hourly` (top of every hour), or a custom 5-field cron expression."

Translate the schedule answer to a cron expression:

| Answer    | Cron          |
|-----------|---------------|
| `daily`   | `0 9 * * *`   |
| `weekly`  | `0 9 * * 1`   |
| `hourly`  | `0 * * * *`   |
| Custom    | use as-is (validate it has 5 space-separated fields) |

Hold the branch name and cron expression for Action 2.

---

## Action 2 — Write `.github/workflows/auto-triage.yml`

Create the directory `.github/workflows/` if it doesn't exist. Then write the file `.github/workflows/auto-triage.yml` with the content below, substituting `<BRANCH>` with the branch from Action 1 and `<CRON>` with the cron expression from Action 1:

```yaml
name: Auto Triage

on:
  schedule:
    - cron: '<CRON>'
  workflow_dispatch:

permissions:
  contents: read
  issues: write

jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: <BRANCH>

      - uses: anthropics/claude-code-action@v1
        with:
          # Use ONE of the following — delete the line you don't use:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          prompt: "Run the /auto-triage slash command."
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## Done

Report what was done:
- `.github/workflows/auto-triage.yml` — `created`, branch `<BRANCH>`, schedule `<CRON>`

Then tell the user:
- Add ONE of these repo secrets and delete the other line from the workflow:
  - API key (any account): `gh secret set ANTHROPIC_API_KEY`
  - OAuth token (Pro / Max subscriber, generated via `claude setup-token`): `gh secret set CLAUDE_CODE_OAUTH_TOKEN`
- Trigger it manually once to confirm it works: `gh workflow run auto-triage.yml`
- After that, it runs on the schedule above.
