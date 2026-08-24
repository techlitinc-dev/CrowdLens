# MT-07 — Monitoring, Alerts & BI

**Prereq:** MT-06 green; Novu + Lago configured (SMTP test email working).

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | Create a monitoring schedule on the MT-02 project (daily cron) | Row in `monitoring_schedules`; next-run time shown in project settings | |
| 2 | Trigger a run manually (`run_now`) | New grounding job executes; diff against previous run computed | |
| 3 | **Shift alert:** in psql, inject a synthetic spike (insert items flipping one theme strongly negative), trigger the detector | Novu `sentiment_shift` email arrives; it names the theme AND links the driver thread. No driver items = no alert (verify by injecting a driverless anomaly → silence) | |
| 4 | Open the narratives view for the project | Clusters are sensible to a human eyeball (read 5); lifecycle stages (emerging/peaking/declining) plausible | |
| 5 | `/dashboards` | Sentiment index, share-of-voice, issue volume, momentum render; per-language cut works (en vs hi); every data point opens the evidence drawer | |
| 6 | Manually trigger the weekly digest | Email arrives: "what changed / why / what to watch"; every claim has a citation link | |
| 7 | **Public API:** create a service token with `read:verdicts` only → `GET` a verdict (expect 200) → `POST` a grounding job (expect 403). Register a webhook; trigger a verdict; verify the signed payload (`X-CrowdLens-Signature` HMAC validates); replay the delivery | All as expected; signature validates with the workspace secret | |
| 8 | **Validation Center:** log a real outcome for the MT-02 verdict (hit or miss) | Entry saved; accuracy page shows the hit rate with n=1 **hidden** (n<10 → no %, just the log) — no fake precision | |
| 9 | **Billing:** after today's cycles, check Lago wallet for the workspace | `decision_cycle` events consumed credits correctly; `cost_ledger` vs Lago drift < 1% | |

**Sign-off:** all PASS → phase 8.
