---
description: Run all 5 mandatory end-of-day steps: PLAN_STATUS + session run log + HITL feedback + Confluence + git commit
---

Run all 5 end-of-day steps in order. Do not skip any step. Do not mark the day done until all 5 are complete.

## Step 1 — Update PLAN_STATUS.md
- Mark today's day row as `[x]` done
- Add a brief notes column entry (what was delivered)
- Update the "Last updated" date at the top to today's date
- Update the "Current Sprint" header if a sprint just completed

## Step 2 — Session Run Log + Metrics Refresh
This ensures Agent Build data is current after every session.

**2a — Write a run log if needed.**
Check `riia-ai-org/agent-ops/runs/` for a run log covering today's session:
- If `/enhance` ran today, the orchestrator already wrote one — skip 2a.
- If today's work was a direct fix or multi-session continuation without a `/enhance` run, write one now:
  `riia-ai-org/agent-ops/runs/run-{YYYYMMDD-HHMM}.json`

  Minimal format for direct/manual work:
  ```json
  {
    "run_id": "{YYYYMMDD-HHMM}",
    "app": "{rita|fno|ops|ds}",
    "request": "{one-line description}",
    "skill_file": "n/a",
    "agents": [{
      "role": "engineer", "status": "pass",
      "steps_required": 3, "steps_completed": 3, "adherence_score": 1.0,
      "token_estimate": 800,
      "grounding_checks": { "branch_created": false, "code_changed": true, "spec_updated": false, "ruff_passed": true, "contract_matches_architect": true },
      "failure_modes": [], "notes": "{what changed, commit hash}"
    }],
    "overall_status": "pass",
    "total_tokens_estimated": 800,
    "duration_minutes": 5,
    "branch": "master", "merge_status": "merged", "merge_commit": "{short hash}"
  }
  ```

**2b — Regenerate metrics.json** (always run after 2a):
```
python riia-ai-org/agent-ops/aggregate_metrics.py
```
Run from workspace root `riia-cowork-jun/`. Note any `[ALERT]` lines in the session summary.

## Step 3 — Collect HITL Feedback
Ask the user for scores on today's session using AskUserQuestion (4 questions, max 4 options each):

1. **Accuracy** (1–5) — was the work correct? Options: 5/4/3/2-or-below
2. **CSAT** (1–5) — overall satisfaction. Options: 5/4/3/2-or-below
3. **Planning OK** — did the agent plan correctly? Options: Yes / No
4. **Time saved** (hours) — Options: <1h / 1–3h / 3–8h / 8+h

Map answers to numbers: 5→5, 4→4, 3→3, "2 or below"→2, "8+"→8, "3–8"→5, "1–3"→2, "<1"→0.5

Write the `human_score` block into the run log created in Step 2:
```json
"human_score": {
  "accuracy": <number>,
  "relevance": 4,
  "csat": <number>,
  "planning_ok": <true|false>,
  "time_saved_hours": <number>,
  "notes": "<any freetext the user added>"
}
```

Re-run `aggregate_metrics.py` after writing scores. If any `[ALERT]` lines appear, note them.

If alerts fire, append to the session summary:
> ⚠ N alert(s) active — run `/agent-performance-improvements` before next session.

## Step 4 — Publish Confluence sprint board
Identify the current sprint number N from PLAN_STATUS.md.
Run: `CONFLUENCE_EMAIL=contact@ravionics.nl python project-office/confluence/sprint-boards/publish_sprint{N}_board.py`
(Run from the workspace root: `riia-cowork-jun/`)
If the script does not exist yet, skip this step and note it as a blocker.

## Step 5 — Git commit
Stage all files changed today. Commit with a message in this format:
```
<type>(<scope>): <what was delivered today>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```
Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

After committing, run `git status` to confirm working tree is clean.

## Done
Report: "Day complete. All 5 end-of-day steps done. Commit: <short hash>"
