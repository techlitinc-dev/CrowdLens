# MT-04 — Simulation Theater & Knowledge Graph

**Prereq:** MT-03 green. Uses the project + completed simulation from MT-02.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | Open `/` home screen | Project cards show status, cost-to-date, confidence chip; dark theme matches tokens (deep slate bg, lime accents) | |
| 2 | Walk `/projects/new` wizard with dummy inputs (don't launch) | ≤5 steps, defaults prefilled, budget cap required | |
| 3 | Launch a NEW simulation (2 variants), open `/projects/:id/simulation` immediately | Live feed animates posts as rounds land; every post has purple SIMULATED badge; branch tabs switch variants; sentiment lanes move per round; round scrubber replays history | |
| 4 | Click a post → ask "Why do you oppose this?" | Answer inline, SIMULATED treatment, run/round context shown | |
| 5 | `/projects/:id/personas`: drag a proportion slider, save | Auto-renormalizes to 1.0; new version created; old version viewable; language samples link to real items | |
| 6 | **Disconnect test:** kill the WS (DevTools offline mode 30s, back online) | Reconnect badge appears; feed resumes without duplicate posts | |
| 7 | `/projects/:id/graph`: search an entity from the verdict, click it, drag the time scrubber, toggle before/after | Focus mode + evidence drawer with real/sim refs; graph changes across time; smooth interaction | |
| 8 | **5K-node perf:** load `tests/fixtures/graph-5k.json` (dev-only route or fixture loader) | Pan/zoom stays fluid — no multi-second freezes | |
| 9 | **Cross-tenant probe:** copy a graph entity URL from project A; open it logged in as a user without access | 403/404, zero data leaked | |
| 10 | **Honesty sweep:** every number on the theater + graph screens | Hovering/clicking opens the evidence drawer; any number that can't → FAIL | |

**Sign-off:** all PASS → phase 5.
