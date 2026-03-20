# multi-agent-collab

A file-based protocol for AI coding agents (Claude Code, Codex, Gemini CLI) to collaborate on a shared codebase with cross-review.

No infrastructure. No sockets. Just markdown files that agents read and write. The filesystem is the message bus.

## How It Works

Two agents share one repo. A `.collab/` directory coordinates them:

- **manifest.md** — who's participating, roles, current status
- **context.md** — shared knowledge that survives across sessions
- **intent-lock.md** — file-level locks to prevent conflicts
- **handoffs/** — "here's what I did, please review"
- **reviews/** — "here's what I found, fix these issues"
- **log.md** — full activity timeline

The human approves every transition. Turn-based — one agent works at a time.

```
Terminal 1 (Claude Code)           Terminal 2 (Codex)
──────────────────────             ──────────────────
Agent A implements                 Agent B waits
Agent A writes handoff    ──→      Agent B reviews
                                   Agent B writes review
Human approves            ──→      Next cycle
```

## Install

**As a Claude Code plugin:**
```bash
claude /plugin install multi-agent-collab@<your-github-username>
```

**Manual:**
```bash
git clone https://github.com/<your-github-username>/multi-agent-collab.git
```

## Quick Start

```bash
# 1. Copy templates into your project
cp -r multi-agent-collab/templates/collab your-project/.collab

# 2. Edit manifest — set your task and agents
vim your-project/.collab/manifest.md

# 3. Copy adapter for each agent
cp multi-agent-collab/adapters/CLAUDE.md.template your-project/CLAUDE.md
cp multi-agent-collab/adapters/AGENTS.md.template your-project/AGENTS.md

# 4. Commit .collab/ so both agents can see it
cd your-project && git add .collab CLAUDE.md AGENTS.md && git commit -m "init collab"

# 5. Open two terminals, start each agent
# Terminal 1: "You are Agent A. Read .collab/manifest.md."
# Terminal 2: "You are Agent B. Read .collab/manifest.md."
```

## Protocol Summary

| Step | Implementer | Reviewer |
|------|-------------|----------|
| 1 | Read `.collab/`, `git pull` | Read `.collab/`, `git pull` |
| 2 | Lock files in `intent-lock.md` | Wait for handoff |
| 3 | Implement, commit, release locks | — |
| 4 | Write handoff, push | Read handoff, `git diff` |
| 5 | Wait for review | Write review, push |
| 6 | — | Wait for human |
| 7 | Human approves → read review → fix if needed | — |

## Supported Agents

| Agent | Adapter | Instruction File |
|-------|---------|-----------------|
| Claude Code | `CLAUDE.md.template` | `CLAUDE.md` |
| Codex CLI | `AGENTS.md.template` | `AGENTS.md` |
| Gemini CLI | `GEMINI.md.template` | `GEMINI.md` |

Any agent that can read/write files and run git commands can participate.

## Repo Structure

```
skills/
  multi-agent-collab/SKILL.md    # Full protocol reference
  cross-review/SKILL.md          # How to review another agent's work
adapters/                        # Per-agent instruction templates
templates/collab/                # Template .collab/ directory
docs/specs/                      # Design specification
```

## License

MIT
