# Pull Request format

Use this structure for PRs opened by automated commands (e.g. `/do-ticket`) and by
hand. Keep it tight — every section should earn its place.

The Jira ticket key MUST appear in **both** the PR title and the body.

## Title

```
<TICKET-ID>: <imperative summary of the change>
```

Example: `SCRUM-1118: add password reset flow`

## Body

```markdown
## Summary
<1–2 paragraphs: what changed and why. Link the ticket: [<TICKET-ID>](<ticket-url>).>

## Changes
- <bullet per notable change, grouped by area (backend / frontend) when full-stack>

## Test plan
- [ ] <how to verify — commands run, scenarios exercised>
- [ ] <acceptance criteria from the ticket, each as a checkable item>

## Notes
<optional: assumptions made, follow-ups deferred, anything a reviewer should know>
```

Do NOT add Claude attribution — no `Co-Authored-By: Claude`, no "Generated with
Claude Code" footer, nothing similar.
