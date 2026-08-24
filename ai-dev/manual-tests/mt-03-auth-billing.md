# MT-03 — Auth, Workspaces & Budget Enforcement

**Prereq:** MT-02 green. Complete the human checklist in `infra/logto-setup.md` FIRST (console work). Two test accounts needed: use two real email addresses you control (Account A = owner, Account B = viewer).

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | Walk `infra/logto-setup.md` checklist | Logto console reachable; app registered; email + Google/GitHub connectors on; org roles owner/analyst/viewer created | |
| 2 | Set `AUTH_DISABLED=false` in `.env`, restart core-api | All `/api/v1/*` endpoints now return 401 without a token | |
| 3 | Browser: open frontend → login as Account A | Redirect to Logto, login succeeds, callback returns, page loads with A's name; DevTools shows JWT attached to API calls | |
| 4 | **Workspace isolation:** as A, create workspace "Client X" + a project. In an incognito window, register/login as B (no membership). As B: `GET /api/v1/workspaces` | B sees zero workspaces; B's direct `GET` on A's project URL → 403/404, not data | |
| 5 | **Invite + role:** as A, invite B as `viewer` to Client X. B accepts (email or invite link) | B now sees Client X; B sees the project but NO "New Project"/launch buttons | |
| 6 | **Role enforcement (server-side, not just UI):** grab B's JWT from DevTools; `curl -X POST .../projects -H "Authorization: Bearer <B_JWT>" -d '{...}'` | `403 FORBIDDEN` — UI hiding is convenience, server is enforcement | |
| 7 | Same curl with A's JWT | Works (owner can create) | |
| 8 | **Virtual key:** as A, create a project with `spend_cap_usd: 2.00`. In LiteLLM UI, find the generated key | Key exists, `max_budget` = 2.00, key stored on the project row | |
| 9 | **Hard cap (the money test):** on that project, run grounding + analysis (enough LLM calls to approach $2). Keep polling project status | When spend crosses ~$2: LiteLLM rejects further calls, project status → `halted_budget`, API surfaces the halt reason. No silent overspend | |
| 10 | **Cap change:** as A, raise cap to $25 via API; rerun step-9 action | Project resumes; key budget in LiteLLM updated to 25 | |
| 11 | **Workspace ceiling:** set workspace cap low; try creating a project whose cap would exceed the remaining workspace budget | `402 BUDGET_EXCEEDED` or validation error — refused | |
| 12 | **Audit log:** perform one of each: login, invite, project create, grounding launch, cap change. Then `GET /api/v1/workspaces/<wid>/audit` as A | 5 matching events with actor, action, target, timestamp | |
| 13 | Same audit endpoint with B's JWT | `403 FORBIDDEN` (owner only) | |
| 14 | **Regression sweep:** re-run MT-02 steps 8–12 with auth ON | Full loop still works with real JWTs; verdict honesty unchanged | |

**Sign-off:** all PASS = MVP cut complete (PBR §12: F-01–F-10 + AUTH-1..4 — "real data in → verdict out, with cost control").
