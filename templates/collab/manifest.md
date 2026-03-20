---
task: "REPLACE_WITH_YOUR_TASK"
created: REPLACE_WITH_DATE
status: active
---

## Agents

| id | agent | role | status |
|----|-------|------|--------|
| A  | REPLACE_AGENT_A | implementer | idle |
| B  | REPLACE_AGENT_B | reviewer | idle |

## Rules
- No agent edits a file locked by the other
- Human approves every review before next cycle
- Roles can swap per-cycle (human decides)
