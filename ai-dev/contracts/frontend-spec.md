# Frontend Spec — Design System & Screen Inventory

**Version 0.1 · React 18 + TypeScript + Vite · Tailwind + shadcn/ui · framer-motion · ECharts · react-force-graph · TanStack Query · react-router**

## Design tokens (fixed — `frontend/src/theme/tokens.css`)

```
--bg:        #0B0F14   (deep slate, dark-first)
--surface:   #121822
--border:    #1F2A36
--text:      #E6EDF3
--muted:     #8B98A5
--accent:    #C6F135   (electric lime — primary actions, verdicts)
--accent-2:  #22D3EE   (cyan — data, links, live indicators)
--danger:    #F45B69
--warn:      #F5B83D
--sim:       #A78BFA   (purple — SIMULATED content, reserved, never reused)
```

Typography: headings `font-serif` editorial (verdicts read like a magazine), data/labels `font-mono`, body sans. Light mode: same tokens inverted — but dark is the default and the demo mode.

## Global UI rules (bind every screen)

1. **Evidence-first:** any number or quote on screen is hoverable/clickable → evidence drawer (right panel) showing the real post (platform, date, link) or sim post (SIMULATED badge in `--sim` purple). No drawer, no number.
2. **Simulated content** always badged SIMULATED, purple, no exceptions — feeds, exports, interviews.
3. **Confidence visible:** verdicts always show agreement score + confidence chip (low/medium/high). No decimal-point theater: no "73.4% will buy" — directions and themes only.
4. **Loading = honest:** a failed collector shows failed, red, with the error — never a spinner that runs forever.
5. **Data-viz-first:** every screen leads with its visual; tables are secondary.
6. Data fetching via TanStack Query only; no ad-hoc `useEffect` fetches. WS via one `useSimulationFeed` hook implementing `contracts/websocket-protocol.md`.

## Screen inventory (PBR §6.2 → routes)

| Route | Screen | Phase |
|---|---|---|
| `/` | Home / Workspaces — project cards: status, cost-to-date, confidence chip | 2 (v0) → 4 (polish) |
| `/projects/new` | Wizard — question → audience → sources → variants → budget cap (≤5 steps) | 2 (v0) → 4 (full) |
| `/projects/:id/grounding` | Grounding Console — collector status, items/source, sample quotes, coverage map | 1 (v0) → 6 (India filters) |
| `/projects/:id/personas` | Persona Panel Editor — archetype cards, proportion sliders, real language samples | 4 |
| `/projects/:id/simulation` | **Simulation Theater** — live feed, round scrubber, archetype sentiment lanes, branch tabs | 4 |
| `/projects/:id/graph` | Knowledge Graph Explorer — search, focus mode, evidence drawer, before/after, time scrubber | 4 |
| `/projects/:id/reports/:rid` | Report Studio — block canvas per `contracts/report-blocks.md` | 5 |
| `/projects/:id/verdict` | Verdict reveal — one-page summary, confidence meter | 2 (v0) → 5 (blocks) |
| `/dashboards` | KPI dashboards — sentiment index, share-of-voice, narrative momentum | 7 |
| `/validation` | Validation Center — prediction-vs-reality log, accuracy page | 7 |
| `/admin` | Keys, budgets, members, audit log, deployment profile | 3 (v0) → 8 (full) |

## Component conventions

- shadcn/ui primitives; custom components in `src/components/` named after domain objects (`VerdictCard`, `ObjectionMap`, `EvidenceDrawer`, `SentimentLane`) — never generic `Card2`, `ChartNew`.
- One chart library: ECharts. Graph: react-force-graph. No others.
- framer-motion for transitions; theater feed animates new posts in (`layout` animations), but motion is disabled under `prefers-reduced-motion`.
- Performance budgets: graph smooth to 5K nodes (viewport culling), theater feed smooth at 20 posts/s (virtualized list), first load < 200 KB JS gzipped per route (code-split by route).

## File layout

```
frontend/src/
├── routes/            # one folder per screen
├── components/        # domain components
├── hooks/             # useSimulationFeed, useProject, ...
├── api/               # typed client generated from api-surface (hand-written types OK)
├── theme/tokens.css
└── lib/               # formatters, evidence-ref resolvers
```
