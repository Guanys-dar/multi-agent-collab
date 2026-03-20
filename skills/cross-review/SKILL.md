---
name: cross-review
description: Use when you are the reviewing agent in a multi-agent collaboration and need to review another agent's code changes
---

# Cross-Review Protocol

You are reviewing another AI agent's code changes. Your review must be thorough, specific, and actionable.

## Process

1. Read the handoff file — understand what was done and what to focus on
2. Run `git pull` to get latest commits
3. Run `git diff <commit-sha>~1..<commit-sha>` to see the actual diff
4. Read changed files in full for context (not just the diff)
5. Write your review to `.collab/reviews/`
6. Append to `.collab/log.md`
7. Set your status to `waiting-for-human` in manifest

## Review Checklist

**Correctness:**
- Does the code do what the handoff says it does?
- Are there logic errors, off-by-ones, or missed edge cases?
- Does it handle errors consistently with the codebase?

**Consistency:**
- Does it follow existing patterns in the codebase?
- Are naming conventions respected?
- Does it match decisions recorded in `.collab/context.md`?

**Completeness:**
- Are all files mentioned in the handoff actually changed?
- Is anything missing that the handoff promised?
- Are there untested paths?

**Risk:**
- Could this break existing functionality?
- Are there security concerns?
- Any performance implications?

## Writing the Review

Use this template:

```markdown
# Review NNN: [Your Agent ID] reviews commit [SHA]
- **verdict:** approved | changes-requested | needs-discussion
- **handoff:** handoffs/NNN-x-to-y.md

## Issues
1. **[blocking]** file:line — what's wrong and how to fix it
2. **[nit]** file:line — suggestion (non-blocking)

## Strengths
- What was done well (be specific)
```

**Verdict rules:**
- `approved` — no blocking issues. Nits are optional to address.
- `changes-requested` — at least one `[blocking]` issue. Must be fixed before proceeding.
- `needs-discussion` — you found something that requires human judgment. Explain clearly.

## Key Rules

- Never approve if you found blocking issues — even under time pressure
- Be specific: file path + line number + what's wrong + suggested fix
- Acknowledge good work in Strengths — collaboration works better with positive signal
- If the handoff is unclear, write `needs-discussion` and explain what's missing
- Read `.collab/context.md` before reviewing — decisions made there are requirements
