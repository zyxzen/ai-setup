Rewrite all commits from `$ARGUMENTS` to HEAD, regrouping them into logical, well-named commits.

The starting commit hash is: $ARGUMENTS

## Workflow

### 1. Validate

- If `$ARGUMENTS` is empty, abort: tell the user the syntax is `/recommit <starting-hash>`.
- Run `git cat-file -e $ARGUMENTS^{commit}` to confirm the hash exists. Abort if it doesn't.
- Confirm the hash is an ancestor of HEAD: `git merge-base --is-ancestor $ARGUMENTS HEAD`. Abort if not.
- Run `git status --porcelain`. If output is non-empty, abort: tell the user to stash or commit first.
- Warn if any commit in `$ARGUMENTS..HEAD` has already been pushed to the remote (`git log $ARGUMENTS..HEAD --not --remotes`). If all commits are already pushed, pause and ask the user whether to continue — this will require a force-push.

### 2. Survey the range

Run the following to understand the scope:

```bash
git log --oneline $ARGUMENTS..HEAD          # list commits in range
git diff --name-only $ARGUMENTS HEAD        # all changed files
git diff --stat $ARGUMENTS HEAD             # summary of changes per file
```

Report the count of commits and files to the user so they can confirm this is the right range.

### 3. Read every changed file

For each file that changed, read:
- Its full diff since `$ARGUMENTS`: `git diff $ARGUMENTS -- <file>`
- Its current state on disk (Read tool)

Build a complete mental model of what changed across the entire range — not commit-by-commit, but holistically.

### 4. Group semantically

Cluster all the changes into a proposed set of logical commits. Guiding principles:

- **One concern per commit** — a commit should do one thing. If the diff adds a model, a policy, a service, and a mutation, those are four separate commits.
- **Tests travel with their subject** unless they're a large standalone sweep (e.g., a spec-only PR). A service spec belongs in the same commit as the service, unless the service already shipped in a previous commit.
- **Migrations and schema changes** are prerequisites — commit them first.
- **Config/initializer changes** get their own commit when they're standalone; otherwise bundle with the thing they configure.
- **Refactors and cleanups** that don't affect behavior get their own commit, separate from feature work.
- Follow conventional commit format: `type(scope): description`
  - Types: `feat`, `fix`, `refactor`, `test`, `chore`, `docs`
  - Subject: imperative mood, max 72 chars, no trailing period
  - Body (optional): explains *why*, not *what*, wraps at 72 chars

### 5. Propose the plan

Present the proposed grouping as a numbered list, showing which files go into each commit:

```
Proposed commits (oldest → newest):

1. chore(db): add pay_schedules migration
   Files: db/migrate/20240101_create_pay_schedules.rb, db/structure.sql

2. feat(models): add PaySchedule model with ClientScoped and soft-delete
   Files: app/models/timesheets_service/pay_schedule.rb
          spec/models/timesheets_service/pay_schedule_spec.rb

3. feat(policies): add PaySchedulePolicy with Pundit scoping
   Files: app/policies/timesheets_service/pay_schedule_policy.rb
          spec/policies/timesheets_service/pay_schedule_policy_spec.rb

4. feat(contracts): add CreatePayScheduleContract
   Files: app/contracts/...
          spec/contracts/...

5. feat(services): add CreatePaySchedule service
   Files: app/services/...
          spec/services/...

6. feat(graphql): expose createPaySchedule mutation and type
   Files: app/graphql/...
          spec/graphql/...
```

Then ask: **"Proceed with this grouping? Reply yes, or describe adjustments."**

Do not proceed until the user confirms.

### 6. Apply

Once confirmed:

1. **Soft-reset** to the starting point — working tree is untouched, everything becomes unstaged:
   ```bash
   git reset --mixed $ARGUMENTS
   ```

2. **For each proposed commit in order:**
   - Stage exactly the files for that commit: `git add <file1> <file2> ...`
   - For files that were deleted, use `git rm <file>`
   - Create the commit using a HEREDOC (multi-line messages only if a body is needed):
     ```bash
     git commit -m "type(scope): description"
     # or multi-line:
     git commit -m "$(cat <<'EOF'
     type(scope): description

     Longer explanation here.
     EOF
     )"
     ```
   - Never add Co-Authored-By, "Generated with Claude", or any attribution.

3. After all commits, verify:
   ```bash
   git log --oneline $ARGUMENTS..HEAD
   ```
   Confirm the count and messages match the plan.

### 7. Report

Show the final `git log --oneline $ARGUMENTS..HEAD` output so the user can see the result.

If any commits in the original range were already pushed, show the exact force-push command but do NOT run it:
```
To update the remote:
  git push --force-with-lease origin <branch>
Coordinate with collaborators before running this.
```

If anything goes wrong mid-apply, stop immediately and tell the user:
- What succeeded so far
- The reflog entry to recover the pre-rewrite state: `git reflog | head -20`

## Safety rules

- Never use `git rebase -i` — it requires a TTY and isn't supported here.
- Never force-push automatically.
- Never skip, squash, or drop commits silently — every change in the range must appear in exactly one proposed commit.
- If a file's changes span multiple concerns (e.g., a file that both fixes a bug and adds a feature), note the ambiguity in the plan and ask the user how to handle it before applying.
