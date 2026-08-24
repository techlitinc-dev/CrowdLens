# MT-01 — Collectors & Grounding

**Prereq:** MT-00 green. Reddit + YouTube keys in `.env`. Pick a test topic with real discussion volume, e.g. `"Nothing Phone 3"`.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | `docker compose up -d bettafish` → `curl localhost:5001/` (or `/health`) | Service responds | |
| 2 | Create a project: `curl -X POST localhost:8000/api/v1/workspaces/<wid>/projects -d '{"name":"Nothing Phone 3 test","question":"Should we launch at ₹34,999?","spend_cap_usd":25}'` | Project created, status `draft` | |
| 3 | Upload seed: `curl -X POST localhost:8000/api/v1/projects/<pid>/seed -F file=@/path/to/seed.md` | Returns `version: 1` and `preview_text` matching the file | |
| 4 | Launch grounding: `curl -X POST localhost:8000/api/v1/projects/<pid>/grounding -d '{"keywords":["Nothing Phone 3"],"sources":["reddit","youtube"],"region":"global","languages":["en"],"time_window_days":30}'` | `job_id` returned; project status → `grounding` | |
| 5 | Poll `GET /api/v1/grounding/<job_id>` every 30s | Progress counters grow; status → `done` within ~30 min; project status → `grounded` | |
| 6 | `docker exec <postgres> psql ... -c "SELECT source, count(*) FROM collected_items GROUP BY source"` | Both sources have >0 rows | |
| 7 | **PII check:** `SELECT text FROM collected_items LIMIT 50` and eyeball; `SELECT count(*) FROM collected_items WHERE text ~* 'u/|@\\w+\\s+(avatar|profile)'` | No usernames, no profile links, no avatars anywhere | |
| 8 | `curl -X POST .../grounding/<job_id>/analyze` then poll `GET /api/v1/reports/baseline/<report_id>` | Report returns; every `themes[].claim` has ≥1 `citation_item_ids` | |
| 9 | **Citation spot-check:** take 3 claims, verify each cited `item_id` exists in `collected_items` and the item's text actually supports the claim | 3/3 real and relevant | |
| 10 | **Negative test:** in psql, tamper one report row: replace a claim's `citation_item_ids` with `["fake_id"]`, set `verified=false`; re-request the report endpoint | `422 INSUFFICIENT_GROUNDING` — unverified reports are never served | |
| 11 | Browser: open `/projects/<pid>/grounding` | Progress shown, sample items table has real links that open real posts, baseline report renders with clickable citations | |
| 12 | **Money check:** LiteLLM spend for the analysis calls; `GET /api/v1/projects/<pid>/costs` | Component `analysis` recorded; total plausibly <$1 | |

**Sign-off:** all PASS (especially 7, 9, 10) → phase 2. Step 9 FAIL means the grounding quality is not good enough to simulate on — fix before proceeding; this is Principle 1.
