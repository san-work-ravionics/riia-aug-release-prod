---
description: Apply targeted improvements to skill files and the /enhance orchestrator based on threshold alerts from aggregate_metrics.py. Run after any /enhance session that produced [ALERT] output.
---

# /agent-performance-improvements — Agent Performance Improvement Loop

Reads current threshold alerts from `metrics.json`, traces each to its root cause, applies targeted fixes to the relevant skill file or orchestrator, updates `skill_version_history`, regenerates metrics, and commits.

**Usage:** `/agent-performance-improvements`  
No arguments required. All inputs are read from `metrics.json` and recent run logs.

---

## How to Execute This Command

Execute each step in sequence. Do not skip steps. Apply fixes only to files that have a confirmed firing alert — do not pre-emptively fix alerts that are not currently breached.

---

## Step 1 — Read Current Alerts

Read `riia-ai-org/agent-ops/metrics.json`.

Extract all firing threshold conditions:

| Condition | Threshold | Source field |
|---|---|---|
| FC code fired too often | any FC code total > 3 | `failure_catalog[].total_fires` or count from run logs |
| Engineer first-pass rate low | `< 0.70` | `skill_version_history[engineer].first_pass_rate` or derive from runs |
| CSAT below threshold | `< 3.5` (non-null only) | `quality.avg_csat` |
| Token forecast error high | `> 35%` | `token_forecasting.avg_forecast_error_pct` |

List every alert that is currently firing. If no alerts are firing, report:
> "No threshold alerts currently firing. Nothing to improve."
Then stop.

---

## Step 2 — Trace Each Alert to Root Cause

For each firing alert, read the relevant recent run logs (last 3 runs in `riia-ai-org/agent-ops/runs/`) to confirm the pattern.

**Alert → Root cause → Fix target mapping:**

| Alert | Root cause | Fix target |
|---|---|---|
| FC-001 ×N (spec not updated) | Spec update rule is a reminder, not a gate — engineers skip it | Skill files: move spec update check before commit; make it a blocking DoD item |
| FC-002 ×N (API contract missing) | Architect output incomplete | `enhance.md`: strengthen Architect validation check |
| FC-003 ×N (ruff failed) | Ruff not run or errors not fixed before commit | Skill files: add explicit ruff gate before `git add` |
| FC-004 ×N (contract mismatch) | JS reads fields not in schema | `enhance.md`: add contract cross-check to QA validation step |
| FC-005 ×N (spec not confirmed by TechWriter) | TechWriter skipped spec check | Skill files: add spec confirmation as required TechWriter step |
| FC-PARTIAL-IMPL ×N | Engineer files-changed < Architect files-to-touch | `enhance.md`: add files-count cross-check before QA advance |
| Engineer first-pass rate < 0.70 | Engineer completing partial implementations | `enhance.md` Step 4 validation: require files_changed ≥ 70% of files_to_touch |
| CSAT < 3.5 | Trace to root — usually partial impl or planning miss | Fix the upstream alert (FC-PARTIAL-IMPL or FC-002) that caused it |
| Token forecast error > 35% | Complexity multipliers miscalibrated | `riia-ai-org/agent-ops/aggregate_metrics.py`: recalibrate multipliers from actual vs forecast data |

Report: one line per alert — "Alert: {alert} → Fix: {what file, what change}"

---

## Step 3 — Apply Fixes

For each alert identified in Step 2, apply the targeted fix. Read the target file first, then edit the specific section.

**Skill file fix pattern (FC-001, FC-003, FC-005):**
- File: `project-office/skills/skill-add-{app}-feature.md` for all 3 apps (rita, fno, ops)
- What to change: in the Engineer DoD checklist or TechWriter section, convert the relevant item from a checklist reminder to an explicit blocking instruction
- Example for FC-001: change `- [ ] Spec updated` to a bolded rule before the commit block: `**STOP — do not run git add until spec is updated. Open the spec file, add the endpoint row, save, then proceed.**`

**Orchestrator fix pattern (FC-PARTIAL-IMPL, FC-002, FC-004, engineer first-pass):**
- File: `.claude/commands/enhance.md`
- What to change: in Step 4 (Engineer validation), add an explicit cross-check:
  ```
  Count files in Architect's "Files to touch" table → N_expected
  Count files in Engineer's "Files changed" list → N_actual
  If N_actual < N_expected × 0.7: report mismatch, ask Engineer to explain gap before advancing to QA
  ```

**Forecast recalibration (token forecast error > 35%):**
- Read the last 5 runs with non-null `token_forecast.total_forecast` and `total_tokens_estimated`
- Compute actual vs forecast ratio per complexity tier
- Update multipliers in `riia-ai-org/agent-ops/aggregate_metrics.py` `by_complexity` section

After each fix, confirm the edit was made and note the before/after rule text.

---

## Step 4 — Update skill_version_history

For each skill file changed, add or update a `skill_version_history` entry in `riia-ai-org/agent-ops/metrics.json`:

```json
{
  "skill_file": "project-office/skills/skill-add-ops-feature.md",
  "last_updated": "<short git hash after commit>",
  "improvement_applied": "FC-001",
  "before_first_pass_rate": <value from metrics before fix>,
  "after_first_pass_rate": null
}
```

`after_first_pass_rate` stays `null` until the next `/enhance` run produces data.

---

## Step 5 — Regenerate Metrics

Run from repo root (`riia-cowork-jun/`):
```
python riia-ai-org/agent-ops/aggregate_metrics.py
```

Note whether any alerts still fire after the changes. If the same alert still fires, it means the fix target was wrong — flag to user.

---

## Step 6 — Commit

Stage all changed files. Commit:
```
fix(agent-ops): improve skill rules for {list of FC codes addressed}

{one line per fix: e.g. "FC-001: spec update made blocking gate in all 3 skill files"}
{e.g. "FC-PARTIAL-IMPL: files-count cross-check added to /enhance Step 4"}

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

Run `git status` to confirm working tree clean.

---

## Final Report

```
── /agent-performance-improvements complete ──────────────
  Alerts addressed: {list}
  Files changed:    {list}
  Alerts still firing after fix: {list or "none"}
  Commit: {hash}

  Next: run /enhance and check if first-pass rate improves.
─────────────────────────────────────────────────────────
```

---

## When to Trigger This Command

The `/enhance` orchestrator should suggest this command at the end of Step 7 if `aggregate_metrics.py` output contains any `[ALERT]` lines. The suggestion text:

> "aggregate_metrics.py flagged {N} alert(s): {list}. Run `/agent-performance-improvements` to address them before the next /enhance run."

This command is not automatically invoked — the user decides when to run it.
