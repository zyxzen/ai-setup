---
description: Review a Jira ticket and publish a light-theme Artifact that explains, section by section, what the ticket asks for and what we need to do to finish it.
argument-hint: <JIRA-TICKET-KEY e.g. SCRUM-1118>
---

You are running the `/explain-ticket` slash command.

# Goal

Read the Jira ticket `$ARGUMENTS` and everything it points to, then publish one HTML
Artifact that tells the reader, in plain language, what the ticket wants, what "done"
looks like, where the work lives, and the steps to get there. The reader should be able
to start work from this page without re-reading the ticket.

This command is read-only. Do not write code, edit the ticket, comment on Jira, move
the ticket, or create branches.

# Input

`$ARGUMENTS` is a Jira issue key like `SCRUM-1118`. Uppercase it and strip whitespace.
If it does not look like `<PROJECT>-<NUMBER>`, stop and tell the user. Below, `<ID>` is
that key.

# Step 1 - Load the theme

Invoke the `ticket-report-artifact` skill with the Skill tool before anything else. It
defines the visual system, the reference build (`example-scrum-2006.html`), the
component classes (`components.md`), the source rule ("every sentence traces to
something you read this session"), and the pre-publish checklist. Follow all of it
except the section list, which this command replaces (Step 4).

# Step 2 - Collect sources (in parallel where possible)

- **Ticket**: `getJiraIssue` with `fields: ["summary","description","status","comment","reporter","assignee","priority","created","updated","labels","issuetype","parent","subtasks","issuelinks","customfield_10020"]`.
  Use whichever Atlassian MCP server is available (`mcp__atlassian-<slug>__*` first,
  then the engineering Atlassian connector, using the cloudId the skill names if it
  applies). If none works, stop and tell the user.
- **Linked context**: parent epic, linked issues and subtasks (summary + status each),
  remote links (`getJiraIssueRemoteIssueLinks`), and any Confluence pages or Figma links
  in the description. Read Confluence pages; for Figma, record the link only unless the
  user asked for design detail.
- **Code** (only if the current directory is a git repo): search for the screens,
  endpoints, models, events or strings the ticket names (`grep -rn`, `git log -S`).
  Record the file paths and what each one does today. Check for an existing branch
  or PR: `git branch -a | grep -i <ID>` and `gh pr list --search <ID> --state all`.
- Keep a list of what you ran and what each showed. It feeds the "Where the work lives"
  section and keeps every claim traceable.

# Step 3 - Work out the answers

Before writing HTML, answer these for yourself from the sources:

1. In one sentence, what does the ticket ask for?
2. **Kind of work**, which picks the variant in Step 4: Bug, Feature, Chore/Refactor,
   or Spike (research). Use the Jira issue type, but override it when the description
   clearly says otherwise (a "Story" that only fixes a crash is a Bug) and say so.
3. What happens today, and what should happen after? For a bug, also the steps to
   reproduce, and whether you could confirm them in the code.
4. **What the comments changed.** Comments often override the description: scope cut,
   a different approach agreed, a question answered. List each change with who and
   when. Where the description and a later comment disagree, the comment wins and
   the page says so.
5. What are the acceptance criteria? Mark each as **Stated** (description), **Agreed**
   (decided in a comment) or **Inferred** (your reading). Never present an inferred
   one as stated.
6. What is in scope, and what is explicitly or obviously out of scope?
7. What must exist or be answered before work starts: blocking linked issues, other
   teams, access, designs, feature flags, migrations, open questions.
8. Which files, services, or screens change? Which stay as they are?
9. What is the order of work? Break it into steps a developer can tick off.
10. How do we test it (automated cases and manual QA steps)?
11. What could go wrong (data, migrations, flags, other teams, performance)?
12. Rough size: Small / Medium / Large, with a one-line reason. Label it an estimate.
13. **Readiness verdict**, one of:
    - **Ready** - criteria are clear and nothing blocks the first step.
    - **Needs answers** - work can start, but at least one open question affects a
      later step.
    - **Blocked** - the first step cannot start until something is answered or done.

# Step 4 - Build the page

Copy the reference build to the scratchpad and keep its `<style>` and `<script>`
unchanged. Replace content using these sections. Include a section only when the
sources support it; renumber and update the contents rail if you drop any.

The page reads top to bottom in the order you would work: understand it, clear what
blocks you, agree what done means, then do it. The first screen alone (hero + 01)
must be enough to know what to do.

| # | Part | Section | Required | Content |
| --- | --- | --- | --- | --- |
| - | - | Hero | yes | Chips: kind of work, Jira status, priority, and the **readiness verdict** (`.chip.pass` Ready, `.chip.review` Needs answers, `.chip.blocked` Blocked; the theme has no blocked chip, so add one rule `.chip.blocked .dot { background: var(--del); }` to the copied style, the only allowed style change). h1 states the task in plain words. 2-sentence lead. Metabar: reporter, assignee, sprint, opened, updated. Before/after pills only if the ticket changes one clear value |
| 01 | Understand | At a glance | yes | `.qa`: "What is this ticket asking?" on grey, plain answer on white. 3 `.fact` cards: Kind of work, Main area, Size (estimate). Then a "Do this" `ol.steps` of 3-5 lines that compresses section 07, each linking to its step there |
| 02 | Understand | Before you start | when anything blocks or is unclear | Merges blockers and open questions. `.question` cards, blocking ones first; the `small` line names who on the ticket can answer it and which step in 07 it holds up. Blocking linked issues, missing access or designs go here too. If nothing is open, drop the section and the hero says Ready |
| 03 | Understand | Today vs after | yes | `.compare`: left panel "Today" (`--old`), right panel "After" (`--new`), in plain words or payloads/UI text. Under it, who is affected, and a `.note` for what is NOT affected. Variant content below |
| 04 | Understand | What changed in the comments | when comments changed scope, approach or criteria | Table: change, who, date, effect on the work. Then the condensed thread per the skill's Discussion rules, skipping comments that changed nothing. Drop the section if the comments only restate the ticket |
| 05 | Plan | Definition of done | yes | Table of acceptance criteria. Source column: `.tag.new` Stated, `.tag.new` Agreed (link the comment in 04), `.tag.old` Inferred. Under it, Out of scope as `ul.unchanged` |
| 06 | Plan | Where the work lives | when in a repo | Table: path, what it does today, what changes. `.code` panels for the key current code. Inline SVG data flow (the skill's graphics rules) only when 2+ systems are involved, with the changing part drawn in `--fix`. Only paths you opened this session |
| 07 | Plan | Step-by-step plan | yes | `ol.log`: each step is one action (`.cmd` holds the file or command) and `p.found` says what it achieves and how you know it worked. Ordered so each step can be merged or tested on its own where possible. Mark a step held up by a question in 02 |
| 08 | Plan | How to test | yes | Test-case table (case, input, expected) and numbered QA steps in `ol.steps`. For a bug, the first case is the reproduction from 03 |
| 09 | Plan | Gotchas | when any | `.risk` cards, one per item that is specific to this ticket (migration, flag, shared component, another team, data backfill). Each says what could go wrong and how to avoid or undo it. No generic risks; drop the section rather than pad it |
| 10 | Appendix | Glossary | when the ticket uses domain terms | `dl.gloss` of every term of art on the page |
| 11 | Appendix | References | yes | `.ref` grid: ticket, parent epic, linked issues, Confluence/Figma links, existing branch or PR, code paths |

**Variants by kind of work** (from Step 3, question 2):

| Kind | Section 03 adds | Section 07 becomes |
| --- | --- | --- |
| Bug | Steps to reproduce as `ol.steps`, expected vs actual, and the likely cause from the code labelled "suspected" until confirmed | Confirm cause, write failing test, fix, verify |
| Feature | The user's path through the change; UI states (empty, loading, error) when a screen is involved; Figma links | Normal build steps |
| Chore/Refactor | What must behave exactly the same afterwards | Steps that keep the code shippable after each one |
| Spike | Replace "After" with "What we need to learn" | "Questions to answer": each with how to find out and what the output is (doc, prototype, estimate) |

Wording rules on top of the skill's:

- Write for someone new to this area of the codebase. Short sentences, no unexplained acronyms.
- If an existing PR or branch is found, add a hero chip for it and say in section 07
  which steps it already covers.
- If the ticket is too thin to plan (no description, no criteria), still publish:
  hero (Blocked or Needs answers), 01, 02, 03, 05 (all Inferred) and 11. Section 02 is
  then the main output.
- `<title>` is a 2-4 word name such as `SCRUM-1118 Export Filters`.

# Step 5 - Check and publish

1. Run the skill's pre-publish checklist, including the grep for banned characters and words.
2. Publish with the Artifact tool: `icon: "checklist"`, a one-sentence `description`.
   Republish to the same path for edits.

# Step 6 - Reply in the terminal

Keep it short:

- The Artifact link, and that it is private until shared.
- The readiness verdict and the task in one sentence.
- The step-by-step plan as a numbered list (titles only).
- Anything in section 02, blocking items first.
- Anything section 04 found that overrides the ticket description.
- Confirm nothing was posted to Jira.
