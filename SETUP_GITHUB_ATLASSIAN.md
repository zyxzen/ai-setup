---
description: Set up per-project GitHub (gh) and/or Atlassian MCP accounts, isolated to the current project
argument-hint: "[gh|atlassian|both]"
---

# Per-project account setup (GitHub `gh` + Atlassian MCP)

Configure the **current project** so the GitHub CLI (`gh`) and/or the Atlassian MCP
server authenticate as a **project-specific account**, isolated from other projects
and from any global/default account. This is machine-agnostic and reusable by anyone.

Requested scope (from the argument, may be empty): **$1**

---

## How this works (the two mechanisms)

- **`gh`** reads its credentials from the directory named by the `GH_CONFIG_DIR`
  env var. Point it at a per-project dir → per-project gh account. Claude Code sets
  that env via the project's `.claude/settings.local.json` (`env` block), which is
  local/gitignored.
- **Atlassian MCP** (a remote OAuth server) stores credentials **keyed by server
  name**, and Claude Code has **no way to disable a user-scope MCP server for a
  single project**. So per-project Atlassian accounts require **uniquely-named,
  local-scope** servers and **no global Atlassian server**. (Two projects sharing
  the same server name share one account — use distinct names for distinct accounts.)

---

## Procedure

1. **Confirm the target project.** Run `pwd`. All per-project config goes in
   `<project>/.claude/settings.local.json` and local-scope MCP entries. If this is
   not a project dir, ask the user to confirm before proceeding.

2. **Determine scope and a slug.** If `$1` isn't `gh`/`atlassian`/`both`, ask the
   user which to set up. Then ask for a short **slug** identifying this account
   (e.g. `work`, `personal`, the org/repo name) — used to name the gh config dir
   and the MCP server. Keep it lowercase, no spaces.

### A. GitHub `gh` per-project account  *(if scope includes gh)*

1. Use gh config dir `~/.config/gh-<slug>` (absolute path).
2. Merge into `<project>/.claude/settings.local.json` (create the file/`env` block
   if missing, never clobber existing keys):
   ```json
   { "env": { "GH_CONFIG_DIR": "<absolute path to ~/.config/gh-<slug>>" } }
   ```
3. **Interactive — tell the user to run it themselves** (you cannot do OAuth/login):
   ```
   GH_CONFIG_DIR=~/.config/gh-<slug> gh auth login
   ```
   They authenticate as the intended account (and choose SSH or HTTPS git protocol).
4. *(Optional, only if this repo pushes over SSH with a specific key)* pin the key
   for this repo only:
   ```
   git config --local core.sshCommand "ssh -i ~/.ssh/<key> -o IdentitiesOnly=yes"
   ```
5. **Verify** (works immediately, with the prefix):
   ```
   GH_CONFIG_DIR=~/.config/gh-<slug> gh api user --jq .login
   ```
   Note: the `settings.local.json` env only loads at **session start**, so the
   *unprefixed* `gh` won't switch accounts until Claude Code is **restarted**.

### B. Atlassian MCP per-project account  *(if scope includes atlassian)*

1. **Ensure "per-project only":** check `claude mcp list` and the top-level
   `mcpServers` in `~/.claude.json`. If a **user-scope** Atlassian server exists,
   per-project accounts are impossible while it lives. With the user's OK, remove it
   (`claude mcp remove <name> -s user`) and re-add it per-project for any other
   project that needs it.
2. **Add a uniquely-named, local-scope server** (run from the project dir):
   ```
   claude mcp add --scope local --transport sse atlassian-<slug> https://mcp.atlassian.com/v1/sse
   ```
3. **Add permissions** to `<project>/.claude/settings.local.json` using the
   `mcp__atlassian-<slug>__` prefix (allow read-only, ask for writes). Merge with any
   existing `permissions`:
   ```json
   {
     "permissions": {
       "allow": [
         "mcp__atlassian-<slug>__getJiraIssue",
         "mcp__atlassian-<slug>__searchJiraIssuesUsingJql",
         "mcp__atlassian-<slug>__getVisibleJiraProjects",
         "mcp__atlassian-<slug>__getTransitionsForJiraIssue",
         "mcp__atlassian-<slug>__getJiraIssueRemoteIssueLinks",
         "mcp__atlassian-<slug>__getJiraProjectIssueTypesMetadata",
         "mcp__atlassian-<slug>__getJiraIssueTypeMetaWithFields",
         "mcp__atlassian-<slug>__getIssueLinkTypes",
         "mcp__atlassian-<slug>__lookupJiraAccountId"
       ],
       "ask": [
         "mcp__atlassian-<slug>__createJiraIssue",
         "mcp__atlassian-<slug>__editJiraIssue",
         "mcp__atlassian-<slug>__transitionJiraIssue",
         "mcp__atlassian-<slug>__addCommentToJiraIssue",
         "mcp__atlassian-<slug>__addWorklogToJiraIssue",
         "mcp__atlassian-<slug>__createIssueLink"
       ]
     }
   }
   ```
4. **Interactive — tell the user** (you cannot do OAuth): run `/mcp`, select
   `atlassian-<slug>`, and log in as the intended Atlassian account.
5. **Verify:** `claude mcp list` should show `atlassian-<slug>: ✓ Connected` after
   login. The `mcp__atlassian-<slug>__*` tools only load at **session start**, so the
   user must **restart** Claude Code before they're usable.

---

## Always tell the user at the end

- **Interactive steps are theirs**: `gh auth login` and the `/mcp` OAuth flow cannot
  be done by the assistant.
- **Restart Claude Code** after setup so `GH_CONFIG_DIR` and the MCP tools load.
- **Validate** any JSON you edit (`python3 -c "import json; json.load(open('<path>'))"`).
- Same server **name** = same Atlassian account across projects; use distinct names
  (e.g. `atlassian-<slug>-be` / `-fe`) when projects need different accounts.
