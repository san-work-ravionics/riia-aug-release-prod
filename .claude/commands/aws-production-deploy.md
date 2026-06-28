---
description: Safe 7-phase production deployment — syncs inner + outer repos, stages changes, pushes to prod, and updates deployment knowledge base
---

You are acting as the **RITA Ops Engineer**. Read `project-office/skills/skill-ops-engineer.md` before proceeding.

Execute all 7 phases in order. Do not skip any phase. Pause and ask the user before any destructive action (force push, overwrite, terraform).

---

## Phase 1 — Pre-flight

**1a. Read the knowledge base.**
Read `project-office/ops-deployments/DEPLOYMENT_KNOWLEDGE.md`. Check the **Active Gotchas** section. Report any active warnings to the user before continuing.

**1b. Check inner repo (dev) status.**
Run: `git status`
- If there are uncommitted changes: list them and ask the user — "Should any of these go into this deployment?"
- If clean: confirm "Dev repo clean."

**1c. Check outer repo (prod) status.**
Run: `git -C riia-jun-release status`
- If there are uncommitted changes: list them and ask the user — "These changes in the prod repo are unstaged. Should I include them in this deployment?"
- If clean: confirm "Prod repo clean."

**1d. Confirm prod remote.**
Run: `git -C riia-jun-release remote -v`
Verify the remote URL contains `san-work-ravionics/riia-jun-release-prod`. If not, stop and alert the user — the prod repo `.git` is pointing at the wrong remote.

**1e. Report pre-flight summary and ask to proceed.**
Output a one-line status for each check. Wait for user to say "proceed" or give instructions before moving to Phase 2.

---

## Phase 2 — Sync Inner Repo (dev)

Run: `git pull origin master`

Report: how many commits were pulled, or "Already up to date."

If the pull fails (merge conflict or diverged history): stop, show the error, and ask the user how to resolve. Do not force-merge.

---

## Phase 3 — Sync Outer Repo (prod)

Run: `git -C riia-jun-release pull origin master`

Report: how many commits were pulled, or "Already up to date."

If there are diverged commits between local and remote:
- Run: `git -C riia-jun-release log --oneline origin/master..HEAD` to show commits that are ahead of remote
- Run: `git -C riia-jun-release diff --stat origin/master...HEAD` to show what changed
- Show the diff to the user and ask: "The prod repo has local commits not yet on remote. Proceed with push?"

---

## Phase 4 — Stage and Commit

**Only run this phase if there are uncommitted changes in the prod repo.**
If the prod repo is clean and already synced, skip to Phase 5.

**4a. Show the diff.**
Run: `git -C riia-jun-release diff --stat`
Show the output to the user.

**4b. Confirm which files to stage.**
Ask the user: "Which files should be included in this commit?" — suggest all changed files by default.
Do NOT run `git add -A` or `git add .` without explicit user approval. Stage specific files only.

**4c. Suggest a commit message.**
Based on the diff, suggest a conventional-commit message in the format:
```
<type>(<scope>): <what this changes>

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```
Types: `feat`, `fix`, `chore`, `docs`, `refactor`

Ask the user to confirm or edit the message before committing.

**4d. Stage and commit.**
Run the approved `git add <files>` commands, then:
```
git -C riia-jun-release commit -m "<approved message>"
```

Run: `git -C riia-jun-release log --oneline -3` to confirm the commit was recorded. Report the short SHA.

---

## Phase 5 — Push to Prod

**5a. Final confirmation.**
Show the user:
```
About to push to: github.com/san-work-ravionics/riia-jun-release-prod (master)
This will trigger GitHub Actions → build Docker image → deploy to EC2.
```
Ask: "Confirm push? (yes/no)"

Only proceed after explicit "yes".

**5b. Push.**
Run: `git -C riia-jun-release push origin master`

Report the output lines (commits pushed, remote refs updated).

If the push is rejected (non-fast-forward): stop. Do NOT suggest `--force`. Show the rejection message and ask the user how to proceed.

**5c. Provide the Actions URL.**
Output:
```
GitHub Actions monitor: https://github.com/san-work-ravionics/riia-jun-release-prod/actions
```
Tell the user to open this link and watch the workflow run. Ask them to report back once it completes (green or red).

---

## Phase 6 — Health Check

Once the user reports the GitHub Actions run has completed:

**If green (success):**
Ask the user to verify the production endpoint is alive by opening:
```
https://riia.ravionics.nl/health
```
Expected response: `{"status": "ok"}`

Also check the main dashboard: `https://riia.ravionics.nl`

Ask: "Health check passed? Site loading correctly?"

**If red (failure):**
Ask the user to paste the failing step's log output.
Cross-reference against all patterns in `project-office/ops-deployments/DEPLOYMENT_KNOWLEDGE.md`.
If a matching pattern is found: show the Fix steps from that pattern.
If no matching pattern: diagnose from the log and walk the user through the fix step by step.
After resolution, push the fix and re-run from Phase 5.

---

## Phase 7 — Post-Deploy Update

**7a. Log the outcome.**
Update `project-office/ops-deployments/DEPLOYMENT_KNOWLEDGE.md`:

- If **successful**: append a row to the **Successful Deploys Log** table:
  ```
  | <YYYY-MM-DD> | <short commit SHA> | <brief description of what was deployed> |
  ```

- If **a new failure occurred**: append a new `### PATTERN-NNN` block to **Known Failure Patterns** following the template at the bottom of `DEPLOYMENT_KNOWLEDGE.md`. Use the next sequential pattern number.

- If an **existing pattern recurred**: increment its `Recurrences:` counter and add a dated note below the fix line.

**7b. If Active Gotchas were added during this session**, check whether they are now resolved and remove them from the Active Gotchas section.

**7c. Report completion.**
Output:
```
Deployment complete.
Commit: <SHA>
Production: https://riia.ravionics.nl
Knowledge base: updated
```

---

## Safety Rules (enforced throughout)

| Rule | Detail |
|---|---|
| Never push to dev repo | All `git push` calls use `git -C riia-jun-release` — the prod repo |
| Never `git add -A` without user approval | Stage named files only |
| Never force push | If push is rejected, stop and ask |
| Never run terraform commands | This command does not touch infrastructure — escalate to the user |
| Never skip Phase 1 knowledge base read | Active Gotchas exist for a reason |
| Never skip Phase 7 | Every deployment — success or failure — must be logged |
