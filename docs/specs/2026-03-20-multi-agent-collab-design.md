# Multi-Agent Collaboration Protocol — Design Spec

**Date:** 2026-03-20
**Status:** Draft
**Author:** Human + Claude Code

## 1. Problem

AI coding agents (Claude Code, Codex, Gemini CLI) each work in isolation. When multiple agents need to collaborate on the same task — one implementing, one reviewing, roles swapping — there is no standard protocol for coordination. Developers manually copy-paste context between terminals and hope agents don't clobber each other's work.

## 2. Solution

A file-based collaboration protocol that any AI coding agent can follow. A `.collab/` directory in the project root acts as the shared coordination layer. Agents read it before acting and write to it after acting. The filesystem is the message bus. No external infrastructure required.

## 3. Design Principles

- **File-based:** All coordination happens through files agents can read/write. No sockets, no IPC, no databases.
- **Agent-agnostic:** Any agent that can read files, write files, and run git commands can participate.
- **Human-gated:** The human approves every review verdict and task transition. Agents propose, human decides.
- **Transparent:** All state is inspectable markdown. The human can open `.collab/` and understand everything.
- **Simulates human collaboration:** The protocol mirrors how two developers share a codebase — through code, reviews, and written context.

## 4. Scope

### In Scope
- 2-agent collaboration (e.g., Claude Code + Codex)
- Same task, different roles (implementer + reviewer, flexible assignment)
- File-level intent locking
- Blocking cross-review with human approval gate
- Shared context document
- Per-agent adapter templates (CLAUDE.md, AGENTS.md, GEMINI.md)

### Out of Scope (future work)
- N>2 agent coordination
- Automatic agent launching/orchestration
- Real-time streaming between agents
- Conflict auto-resolution

## 5. Directory Structure

```
.collab/
  manifest.md              # Agent roster, roles, task description
  context.md               # Shared understanding, decisions, open questions
  intent-lock.md           # File-level intent declarations before editing
  log.md                   # Append-only activity timeline
  reviews/
    001-<reviewer>-<sha>.md   # Cross-review of a specific commit
  handoffs/
    001-<from>-to-<to>.md     # Handoff with context for next agent
```

## 6. Protocol Components

### 6.1 Manifest (`manifest.md`)

The entry point. Human creates it to initialize collaboration.

```markdown
---
task: "Implement user authentication with JWT"
created: 2026-03-20
status: active
---

## Agents

| id | agent | role | status |
|----|-------|------|--------|
| A  | claude-code | implementer | idle |
| B  | codex | reviewer | idle |

## Rules
- Agent A implements, Agent B reviews
- No agent edits a file locked by the other
- Human approves every review before next cycle
- Roles can swap per-cycle (human decides)
```

**Fields:**
- `id`: Short identifier (A/B) for log entries and references
- `agent`: Which CLI tool (claude-code, codex, gemini)
- `role`: Current role (implementer, reviewer, or custom)
- `status`: `idle`, `working`, `reviewing`, `waiting-for-human`

### 6.2 Shared Context (`context.md`)

The shared memory between agents. Both read before starting, both append when they learn something.

```markdown
# Shared Context

## Task Understanding
[Human or agent writes initial understanding of the task]

## Codebase Notes
- [timestamp Agent X] Key file: src/index.ts — app entry point
- [timestamp Agent Y] Tests use vitest, co-located as *.test.ts

## Decisions Made
- [2026-03-20 Human] Access token TTL: 15min, refresh token: 7 days
- [2026-03-20 Agent A] Using jsonwebtoken (already in package.json)

## Open Questions
- [2026-03-20 Agent B] Should refresh tokens go in DB or Redis?
```

**Rules:**
- `Codebase Notes` and `Decisions Made` are append-only — never delete entries
- `Open Questions` can be resolved by moving to `Decisions Made` (human does this)
- Every entry is timestamped and attributed
- Agents MUST read this before every work cycle

### 6.3 Intent Locking (`intent-lock.md`)

Prevents two agents from editing the same file.

```markdown
# Intent Lock

| file | agent | action | declared |
|------|-------|--------|----------|
| src/routes/auth.ts | A | create | 2026-03-20 14:30 |

## Released
| file | agent | released |
|------|-------|----------|
| src/index.ts | A | 2026-03-20 14:25 |
```

**Protocol:**
1. Before editing a file, agent reads `intent-lock.md`
2. If file is locked by other agent → do NOT edit. Write to `Open Questions` in `context.md` and set own status to `waiting-for-human`
3. If unlocked → add row to active table, then edit
4. After committing → move row to `Released` section
5. Stale lock rule: if lock is >1 hour old and owning agent's status is `idle`, the other agent flags it in `context.md`. Only the human clears stale locks.

### 6.4 Activity Log (`log.md`)

Append-only timeline for human oversight.

```markdown
# Activity Log

| time | agent | action | detail |
|------|-------|--------|--------|
| 14:30 | A | lock | src/routes/auth.ts |
| 14:45 | A | commit | abc1234 — login/logout endpoints |
| 14:45 | A | handoff | handoffs/001-a-to-b.md |
| 14:50 | B | review | reviews/001-b-abc1234.md → changes-requested |
| 14:55 | human | approve | review 001 accepted |
```

Every agent appends a row after any action. Human can scan the full session history.

### 6.5 Handoffs (`handoffs/`)

When an agent finishes a unit of work, it writes a handoff for the next agent.

```markdown
# Handoff 001: Agent A → Agent B

- **commit:** abc1234
- **files changed:** src/routes/auth.ts, src/middleware/auth.ts
- **status:** ready-for-review

## What I Did
[Summary of changes]

## What To Review
[Specific areas to focus on, line references]

## What's Next
[What the implementing agent plans to do after review passes]
```

**Naming:** `handoffs/NNN-<from>-to-<to>.md` — sequential, direction explicit.

### 6.6 Cross-Reviews (`reviews/`)

The reviewing agent reads the handoff, inspects the diff, and writes a review.

```markdown
# Review 001: Agent B reviews commit abc1234

- **verdict:** changes-requested | approved | needs-discussion
- **handoff:** handoffs/001-a-to-b.md

## Issues
1. **[blocking]** file:line — description of issue
2. **[nit]** file:line — minor suggestion

## Strengths
- [What was done well]
```

**Verdict values:**
- `approved` — changes are good, proceed
- `changes-requested` — blocking issues found, implementer must fix
- `needs-discussion` — escalate to human for a decision

**Review cycle:**
1. Agent B reads handoff → reads `git diff <sha>` → writes review
2. Human reads review → approves or adds comments
3. If `changes-requested`: Agent A reads review, fixes, commits, writes new handoff
4. Repeat until `approved`
5. Human must explicitly approve before workflow advances

## 7. Agent Lifecycle

```
Agent reads .collab/manifest.md
  → checks own role and status
  → reads .collab/context.md
  → reads .collab/intent-lock.md
  → reads .collab/log.md

If role = implementer and status = idle:
  → declare intent on files to edit
  → implement changes
  → commit
  → write handoff
  → append to log
  → update own status to "waiting-for-human"

If role = reviewer and handoff exists:
  → read handoff
  → read git diff
  → write review
  → append to log
  → update own status to "waiting-for-human"

Human approves → cycle advances
```

## 8. Two-Terminal Workflow

The expected runtime: human has two terminal windows open, one per agent.

```
Terminal 1 (Claude Code)           Terminal 2 (Codex)
──────────────────────             ──────────────────
Human: "implement auth"            Human: "you are Agent B,
                                    read .collab/ and wait"

Agent A reads .collab/              Agent B reads .collab/
Agent A locks files                 Agent B: idle (no handoff)
Agent A implements, commits
Agent A writes handoff/001

                                    Human: "check for handoff"
                                    Agent B reads handoff/001
                                    Agent B reads git diff
                                    Agent B writes review/001

Human reads review
Human (T1): "fix issues in
.collab/reviews/001"
Agent A reads review, fixes
Agent A writes handoff/002
...cycle continues...
```

The filesystem is the message bus. The human is the router.

## 9. Per-Agent Adapters

The plugin ships adapter templates that users drop into their project root.

**CLAUDE.md adapter:** Instructs Claude Code to follow the `.collab/` protocol using Read/Write/Edit tools.

**AGENTS.md adapter:** Same protocol adapted for Codex's `read_file`/`apply_patch`/`cmd` tools.

**GEMINI.md adapter:** Same protocol for Gemini CLI.

Each adapter is short (<50 lines) — it tells the agent:
1. Read `.collab/` before any action
2. Follow intent-lock rules
3. Write handoffs/reviews after actions
4. Append to log after everything

## 10. Plugin Repo Structure

```
multi-agent-collab/
  .claude-plugin/
    plugin.json                # Plugin metadata
  package.json                 # Version info
  LICENSE
  README.md
  skills/
    multi-agent-collab/
      SKILL.md                 # Main skill — full protocol reference
    cross-review/
      SKILL.md                 # How to review another agent's work
  adapters/
    CLAUDE.md.template         # Drop into project for Claude Code
    AGENTS.md.template         # Drop into project for Codex
    GEMINI.md.template         # Drop into project for Gemini CLI
  templates/
    collab/                    # Template .collab/ directory
      manifest.md
      context.md
      intent-lock.md
      log.md
      reviews/.gitkeep
      handoffs/.gitkeep
```

## 11. Success Criteria

- Two agents can collaborate on a shared task without file conflicts
- Every code change gets cross-reviewed before the workflow advances
- Human can understand the full collaboration state by reading `.collab/`
- Protocol works with Claude Code + Codex out of the box
- Setup takes <2 minutes (copy templates, fill in manifest)
