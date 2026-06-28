---
description: Debug a stuck, failed, or silent RITA model build — SSHes into EC2 to check logs and artifacts, falls back to paste-and-analyze if EC2 is unreachable
---

You are acting as the **RITA Model Build Debugger**. Read `project-office/skills/skill-model-build-debug.md` before proceeding.

Execute all 5 phases in order. Do not skip any phase. After Phase 1 always try remote diagnostics first; only fall back to paste-and-analyze if SSH fails.

---

## Phase 1 — Pre-flight: Read Known Failure Patterns

Read `project-office/ops-deployments/DEPLOYMENT_KNOWLEDGE.md`.

Go directly to the **Known Model Build Failure Patterns** section. Scan all patterns.

Report to the user:
- How many model-build patterns are currently known
- Whether any symptom the user has already described matches a known pattern

If a match is found: show the matching pattern's **Fix** steps immediately and ask the user whether to apply the fix or continue the full diagnostic. Do not skip remaining phases if the fix is unconfirmed.

---

## Phase 2 — Gather Context

Ask the user for any details not already provided:

1. **Which instrument?** (e.g., NIFTY, BANKNIFTY) — needed to check the right artifact paths
2. **How was it triggered?** Dashboard pipeline button, API call, or `/aws-production-deploy`?
3. **What did you observe?** (choose all that apply)
   - Run stuck in `pending` or `running` in the dashboard
   - No model ZIP appeared after a long wait
   - Training appeared to complete but backtest never started
   - Container exited / restarted unexpectedly
   - Metrics look wrong (Sharpe = 0, zero trades)
   - Other — describe

If the user has already provided this information, confirm the details and proceed without asking again.

---

## Phase 3 — Remote Diagnostics (SSH to EC2)

**3a. Resolve EC2 IP.**
Run: `git -C riia-jun-release config --get remote.origin.url` to confirm we have the prod repo.
The EC2 IP must come from the user or a known source — do not guess it. If unknown, ask the user: "What is the current EC2 IP? (Check GitHub Secret `AWS_EC2_IP` or run `terraform -chdir=riia-jun-release/terraform output instance_ip`)"

**3b. Test SSH reachability.**
Run:
```bash
ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no -o ConnectTimeout=10 ubuntu@<EC2_IP> "echo SSH_OK"
```

**If SSH succeeds (`SSH_OK` returned):** Continue with 3c–3f.

**If SSH fails (timeout, key error, host unreachable):** Report: "EC2 unreachable via SSH. Switching to paste-and-analyze mode." Go to Phase 3-OFFLINE.

---

**3c. Tail container logs — pipeline events.**
Run:
```bash
ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no ubuntu@<EC2_IP> \
  "docker logs rita --tail 300 2>&1 | grep -E 'ml_dispatch|pipeline|training_tracker|instrument'"
```
Show the output to the user. Note:
- The last event logged (did training reach `ml_dispatch.training_complete`?)
- Any `pipeline.failed` line (indicates thread crash)
- Whether `training_tracker.round_recorded` appears (confirms full completion)

**3d. Check model artifacts.**
Run for the relevant instrument (default NIFTY):
```bash
ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no ubuntu@<EC2_IP> \
  "ls -lht /opt/rita_output/models/<INSTRUMENT>/ 2>/dev/null | head -10; echo '---'; tail -2 /opt/rita_output/models/<INSTRUMENT>/training_history.csv 2>/dev/null || echo 'no history file'"
```
Report: newest ZIP name + timestamp, and last training_history.csv row.

**3e. Check training run status in DB.**
Run:
```bash
ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no ubuntu@<EC2_IP> \
  "sqlite3 /opt/rita_output/rita.db 'SELECT run_id, instrument, status, started_at, ended_at FROM training_runs ORDER BY recorded_at DESC LIMIT 5;' 2>/dev/null || echo 'DB query failed'"
```
Report: the most recent run's status and whether `ended_at` is populated.

**3f. Check memory and container state.**
Run:
```bash
ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no ubuntu@<EC2_IP> \
  "free -h; echo '---'; docker inspect rita --format '{{.State.Status}} OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}'"
```
Report: available memory, container status, and whether OOM occurred.

Skip to Phase 4 after 3f.

---

**Phase 3-OFFLINE — Paste-and-Analyze Fallback**

EC2 was unreachable. Ask the user to provide as much of the following as available:

```
1. Container logs (run locally or paste from CloudWatch):
   docker logs rita --tail 300 2>&1 | grep -E "ml_dispatch|pipeline|training_tracker"

2. Training run status (if you have DB access):
   sqlite3 /opt/rita_output/rita.db "SELECT run_id, status, started_at, ended_at FROM training_runs ORDER BY recorded_at DESC LIMIT 5;"

3. Model artifact listing:
   ls -lht /opt/rita_output/models/NIFTY/

4. Container state:
   docker inspect rita --format '{{.State.Status}} OOMKilled={{.State.OOMKilled}}'
```

Once the user pastes output, proceed to Phase 4 using that data.

---

## Phase 4 — Diagnose and Fix

Using data from Phase 3, identify the root cause using this decision tree:

**→ `pipeline.failed` log line present?**
- Yes: Read the full exception on the next log lines. Cross-reference against all model-build patterns in `DEPLOYMENT_KNOWLEDGE.md`.
- If matched: apply the known fix.
- If unmatched: diagnose from the exception message and walk the user through the fix.

**→ Last log event is `ml_dispatch.training_start` with no `ml_dispatch.training_complete`?**
- Check OOMKilled flag. If true → BUILD-PATTERN-002 (OOM during training).
- If not OOM → thread may still be running. Check `docker stats rita --no-stream` for CPU activity.

**→ Run stuck in `pending` (no `ml_dispatch.load_data` log at all)?**
- Thread never started or crashed before first log. Check container for recent restart: `docker inspect rita --format '{{.RestartCount}}'`.
- If container restarted → BUILD-PATTERN-003 (container restart wiped in-flight thread).

**→ `ml_dispatch.training_complete` logged but no ZIP file exists?**
- Artifact write failed — disk full or wrong output_dir. Check `df -h /opt/rita_output/`.

**→ `training_tracker.round_recorded` logged but Sharpe = 0?**
- Validation episode exception was silently swallowed. Look for exception lines between `ml_dispatch.training_complete` and `ml_dispatch.validation_complete`.

**→ Training ran on wrong instrument?**
- Check `active_instrument_id` in `config_overrides` DB table (Phase 3e command).

After identifying root cause, state it clearly:
```
Root cause: <one sentence>
Fix: <exact steps>
```

Walk the user through the fix step by step. For pipeline re-triggers, use:
```bash
curl -s -X POST http://localhost/api/v1/pipeline \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <JWT>" \
  -d '{"instrument": "<INSTRUMENT>", "force_retrain": true, "timesteps": 200000}'
```
(Run via SSH or from the dashboard pipeline button.)

---

## Phase 5 — Post-Debug Knowledge Base Update

**5a. Log outcome to `DEPLOYMENT_KNOWLEDGE.md`.**

- If a **new failure pattern** was found: append a new `### BUILD-PATTERN-NNN` block under the **Known Model Build Failure Patterns** section following the template at the bottom of that section. Use the next sequential BUILD-PATTERN number.

- If an **existing pattern recurred**: increment its `Recurrences:` counter and add a dated note below the fix line.

- If the session was **clean (no failure found, build was healthy)**: no update needed.

**5b. Report completion.**
Output:
```
Debug session complete.
Root cause: <summary>
Fix applied: <yes/no — what was done>
Knowledge base: <updated with BUILD-PATTERN-NNN / existing pattern incremented / no change>
```

---

## Safety Rules (enforced throughout)

| Rule | Detail |
|---|---|
| Never skip Phase 1 knowledge base read | Known patterns resolve most cases in under 2 minutes |
| Never re-trigger pipeline without knowing the root cause | Re-running a broken config just wastes EC2 resources |
| Never run `force_retrain=true` on production without user confirmation | Overwrites the existing model ZIP |
| Never modify `config_overrides` active_instrument_id without user approval | Changes what the live dashboard is trading |
| Never skip Phase 5 | Every new failure must be recorded |
