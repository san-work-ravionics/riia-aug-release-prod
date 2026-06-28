---
description: Run a scoped engineer task with all RITA rules inline — no spec file reads needed
---

You are an Engineer agent for the RITA production codebase.

**Task:** $ARGUMENTS

---

## Step 1 — Choose the right command

Run the specialized command that matches this task — it contains all rules and the definition of done:

| Task type | Use command |
|---|---|
| Add or modify a FastAPI endpoint | `/add-endpoint <task>` |
| Fix a frontend JS bug | `/fix-bug <bug description>` |
| Add a DB model, repository, or migration | `/add-db-model <task>` |
| Add a chat classifier intent | `/add-chat-intent <task>` |
| Add/update a feature in the RITA dashboard | `/add-rita-feature <task>` |
| Add/update a feature in the FnO dashboard | `/add-fno-feature <task>` |
| Add/update a feature in the Ops dashboard | `/add-ops-feature <task>` |
| Add a new data field, analysis, or ML model | `/add-data-feature <task>` |
| Add/update a feature in the Mobile PWA | `/add-mobile-feature <task>` |

If none of the above match, continue below with the general guardrails.

---

## General Guardrails (when no specialized command applies)

**Architecture:**
- Routes go in the correct tier: system (`api/v1/system/`), workflow (`api/v1/workflow/`), or experience (`api/experience/`)
- No direct DB/file I/O in routes or services — all data access via `repositories/` only

**Session/DB:**
- Every repo constructor requires `db: Session` — never call `MyRepo()` without it
- FastAPI dependency: `def get_svc(db: Session = Depends(get_db)) -> MyService: return MyService(db)`
- Background threads must open their own session via `SessionLocal()` and close in `finally`
- `upsert()` already calls `db.commit()` — do not commit again

**Code quality:**
- No `print()` statements — use `structlog`
- No hardcoded lot sizes — NIFTY=75, BANKNIFTY=30 must come from `settings.instruments.*`
- No external API calls — all data is local CSV/SQLite
- Do not touch `rita_input/` — read-only source data

**Spec maintenance:**
- Any change to an API contract, schema, or data layout → update `Specs/Spec_Python_Code.md` in the same commit

---

## Step 2 — Implement

Work in targeted slices (max 400 lines per file read). Implement only what the task requires.

---

## Minimum Definition of Done

- [ ] `ruff check src/` passes
- [ ] New endpoint: JS field list matches handler's `return` dict exactly
- [ ] `Specs/Spec_Python_Code.md` updated if API contract changed
