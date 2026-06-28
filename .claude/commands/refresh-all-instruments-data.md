---
description: Refresh all instruments' price data from yfinance and update the DB cache
---

This command refreshes OHLCV data for all 11 RITA instruments by fetching delta rows from yfinance.

Steps:
1. Check the server is running: use the Bash tool to run `curl -s http://localhost:8000/health` and confirm it returns ok.
2. Call the refresh endpoint: use the Bash tool to run:
   `curl -s -X POST http://localhost:8000/api/v1/instrument/refresh-all | python -m json.tool`
3. Parse and display the results as a formatted table showing: Instrument | Gap Days | Raw Rows Added | DB Rows Inserted | Status
4. Report the total: "{refreshed} instruments refreshed, {already_current} already current"
5. If any instrument shows status "error", display the error message and suggest re-running.

6. **Refresh production data** (safe — no redeploy, no model changes, no container restart):
   a. SSH into the EC2 prod instance and call the same refresh endpoint inside the running container:
      ```bash
      ssh -i riia-jun-release/terraform/generated-key.pem -o StrictHostKeyChecking=no -o ConnectTimeout=10 ubuntu@13.206.230.76 "curl -s -X POST http://localhost:8000/api/v1/instrument/refresh-all"
      ```
   b. If SSH fails (timeout / connection refused), report "Production refresh skipped — EC2 unreachable" and continue.
   c. If the curl succeeds, parse the JSON response and display the same formatted table as Step 3, prefixed with **"Production:"**.
   d. If any prod instrument shows status "error", display the error and suggest re-running.
