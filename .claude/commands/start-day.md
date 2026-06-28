---
description: Read PLAN_STATUS.md and report today's tasks, status, and blockers
---

Read `PLAN_STATUS.md` (the full file — it is the single source of truth for sprint progress).

Report the following in a concise table:
1. Current day number and sprint
2. Today's tasks — each with role, description, and status ([x] done / [ ] pending)
3. Any blockers listed
4. Overall project completion %

Then identify, for each pending task, which command to use:

| Task type | Command |
|---|---|
| Add or modify a FastAPI endpoint | `/add-endpoint <task>` |
| Fix a frontend JS bug | `/fix-bug <bug description>` |
| Add a DB model or repository | `/add-db-model <task>` |
| Add a chat classifier intent | `/add-chat-intent <task>` |
| Add/update a feature in the RITA dashboard | `/add-rita-feature <task>` |
| Add/update a feature in the FnO dashboard | `/add-fno-feature <task>` |
| Add/update a feature in the Ops dashboard | `/add-ops-feature <task>` |
| Add a new data field, analysis, or ML model | `/add-data-feature <task>` |
| Add/update a feature in the Mobile PWA | `/add-mobile-feature <task>` |
| General engineer task | `/engineer-task <task>` |
| End-of-day routine | `/end-day` |

Finally ask: "Which task should I start?" — do not start any work until the user confirms.
