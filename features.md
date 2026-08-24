# CrowdLens — Feature Catalog

**Version:** 0.1 (Draft) · **Date:** 2026-08-25 · **Companion to:** CrowdLens_PBR.md v0.1, user-stories.md v0.1

The complete feature list, consolidated from PBR §5 and §7. IDs are stable — stories (user-stories.md), roadmap milestones (PBR §13), and cost config (PBR §11.1) all reference them.

**Tiers:** P0 = MVP · P1 = Productization · P2 = Expansion · P3 = Frontier (simulation & AI depth)

**Global rule (PBR §2):** every feature that emits a claim must keep it traceable to a real citation or a labeled simulation run; verdicts require ensemble convergence; no invented magnitudes.

**Scope:** the platform is domain-agnostic — one pipeline (ground → simulate → verdict) serves all scenarios: marketing, product, pricing, PR/crisis, policy, content, hiring, investor comms, local business, personal decisions (full matrix in PBR §7). Only F-13 is advertising-specific.

---

## P0 — MVP

*"Real data in → verdict out, with cost control."*

| ID | Feature | Description |
|---|---|---|
| F-01 | Projects & workspaces | Multi-tenant project containers with member, role, key, and budget isolation per workspace. Multi-client separation for agencies. |
| F-02 | Seed document upload | The artifact under test (PDF/MD/TXT) — concept note, landing page, policy draft. Versioned per project. |
| F-03 | Grounding collectors | Real-data collection from Reddit (PRAW), YouTube Data API v3, X (official/licensed provider), Google News/GDELT, Telegram public channels. Official APIs first. |
| F-04 | Baseline sentiment report | The "what the world thinks now" layer. Every claim carries ≥1 real citation (platform, date, link); weak-evidence claims are flagged, not smoothed. |
| F-05 | Auto persona panel | Audience archetypes with proportions, stances, and real language samples derived from grounding data. Fully editable before simulation. |
| F-06 | Counterfactual branching | Each decision variant spawns an independent simulation branch from the same grounded seed. Up to 5 variants; MVP cap 3 per §11.1. |
| F-07 | Ensemble runs + convergence scoring | Adaptive ensemble: starts at 3 runs, extends until ≥80% agreement, caps at 7. Non-convergent decisions return "no verdict" with divergence analysis. |
| F-08 | Cost estimate + hard spend cap | Upfront estimate before launch; project halts at cap. Enforced by LiteLLM per-project virtual keys (US-3.5). |
| F-09 | Verdict report v1 | One-page executive verdict: ranked options, top objections with quotes, risk flags, confidence + convergence scores. |
| F-10 | Live simulation view + agent interview | Real-time feed of agents posting/arguing, round scrubber, per-archetype sentiment lanes; `/ask` lets you interview any agent (answers labeled as simulated). |

## Authentication & Accounts (AUTH)

| ID | Requirement | Tier |
|---|---|---|
| AUTH-1 | Email + password, magic link, Google/GitHub social login | P0 |
| AUTH-2 | Multi-tenant workspaces with member invites | P0 |
| AUTH-3 | RBAC: Owner / Analyst / Viewer per workspace | P0 |
| AUTH-4 | Per-project LiteLLM virtual keys; spend caps per project and workspace | P0 |
| AUTH-5 | SSO (SAML/OIDC) for Enterprise | P1 |
| AUTH-6 | Personal API keys + scoped service tokens for the public API | P1 |
| AUTH-7 | Audit log of all data access and report exports | P1 |
| AUTH-8 | On-prem license-key activation, offline token refresh window | P1 |

---

## P1 — Productization

| ID | Feature | Description |
|---|---|---|
| F-11 | Interactive Report Studio | Living reports: ordered blocks (verdict card, timelines, sim comparison, persona panel, graph embeds, objection map, evidence rail, cost ledger). Ask-the-report chat, branch what-ifs, live mode, share links, white-label export (PBR §8). |
| F-12 | Knowledge Graph Explorer | Neo4j entity graph with search, focus mode, evidence drawer, before/after toggle, time scrubber. Smooth to 5K nodes. |
| F-13 | Ad Testing module | One vertical module among many — the platform itself is domain-agnostic (PBR §7 preamble). Up to 10 creative variants, per-segment verdicts, objection quotes, live-test result upload for calibration. Direction and objections only — no invented CTRs. |
| F-14 | India data pack | YouTube India (Hindi/Marathi/regional), Reddit India, Telegram public channels, app-store reviews, e-commerce reviews via providers (PBR §9.2). |
| F-15 | Multilingual analysis | Marathi/Hindi/Hinglish content handling; reports in EN + HI/MR. |
| F-16 | Continuous monitoring & alerts | Scheduled collectors, sentiment-shift alerts via Novu (WhatsApp/email), each alert linking to the driving thread. |
| F-17 | Scheduled re-simulation | "Re-run with last week's data" subscriptions; re-ground then re-simulate, report diffs highlighted. |
| F-18 | Validation corpus + public accuracy page | Langfuse datasets; prediction-vs-reality log; published backtests with methodology — the enterprise sales asset. |
| F-19 | White-label agency exports | Client-branded PDF/PPTX; view-only expiring watermarked share links with per-link analytics. |
| F-20 | Public API + webhooks | Embedding for agency/enterprise stacks; events for grounding done, sim done, verdict issued. |
| F-27 | Agent memory & relationship graph | Agents remember interactions across rounds; influence accumulates through persistent social ties, not per-round resets. |
| F-28 | Seeded influencer agents | High-reach voices detected in grounding data become high-centrality agents with their real style and stance. |
| F-29 | Mid-simulation event injection | Scripted shocks at chosen rounds (viral post, price change, news event) — crisis and resilience testing. |
| F-39 | Narrative tracking & momentum | Posts clustered into narratives, each tracked through its lifecycle (emerging → peaking → declining) with a momentum score. |
| F-40 | Anomaly detection & crisis early-warning | Statistical spike detection on all KPIs; composite crisis score with drill-down to the posts driving it. |
| F-41 | Automated insight digests | Scheduled plain-language briefings per project: what changed, why, what to watch. Delivered via Novu. |

---

## P2 — Expansion

| ID | Feature | Description |
|---|---|---|
| F-21 | On-prem / air-gapped deployment | Single docker-compose / Helm profile; local models via vLLM/Ollama; GPU sizing tiers 24GB/48GB/2×80GB; license keys. |
| F-22 | Vertical template marketplace | Prebuilt packs: PR crisis, course launch, product design, dating, campaign intelligence, restaurant/local business, YouTube growth. |
| F-23 | Client ground-truth upload | Surveys via Formbricks, recordings transcribed via Whisper → auto-validation and per-tenant calibration. |
| F-24 | Aspect-based sentiment layer | PyABSA in verdicts — "loved the price, hated the name." |
| F-25 | Collaboration | Comments on reports, shared workspaces, approval flows. |
| F-26 | Consumer self-serve mini-tool | Dating-profile/consumer tool as viral acquisition funnel. |
| F-30 | Adversarial stress-testing | Devil's-advocate agent swarms attack the leading option; a verdict must survive the challenge round to ship. |
| F-31 | Longitudinal simulation | Multi-week runs with scheduled re-grounding; per-variant sentiment drift curves over time. |
| F-32 | Synthetic-survey calibration | Agent opinion distributions benchmarked against real poll/survey data before a run is accepted; failed calibration blocks verdict issuance. |
| F-42 | Custom KPI builder | Calculated metrics and workspace-composed dashboards on grounding + simulation data. |
| F-43 | BI connectors | Native Looker / Tableau / Power BI / Google Sheets connectors atop the public API (F-20). |
| F-44 | Segment & cohort analytics | Cuts by region, language, persona archetype; cohort trend comparison over time. |
| F-45 | Benchmark indices | Opt-in, anonymized cross-tenant benchmarks — share-of-voice and sentiment norms per vertical. |

---

## P3 — Frontier (Simulation & AI Depth)

Longer-horizon bets that deepen the simulation core. All remain bound by the Product Principles — grounded, ensemble, honest-confidence.

| ID | Feature | Description |
|---|---|---|
| F-33 | Counterfactual tree search | The engine proposes the decision space itself: auto-generates variant branches, explores, and prunes low-signal ones within budget. |
| F-34 | Narrative trajectory forecasting | Theme-emergence prediction over 2–8 week horizons; every forecast logged and scored against the validation corpus. |
| F-35 | Archetype-distilled models | Fine-tuned small models per persona archetype to cut ensemble run cost without changing agent distributions. |
| F-36 | Generative focus groups | Live moderated sessions with an agent subset; the analyst asks follow-ups in real time, answers carry sim-run labels. |
| F-37 | Multi-public simulation | Linked simulations across regions/languages; narratives migrate between publics (e.g. English-urban → Hindi tier-2/3). |
| F-38 | Auto-calibration loop | Logged verdict outcomes continuously tune persona proportions; human-approved updates, full drift audit trail. |
| F-46 | Simulated leading indicators | Simulation-derived early-warning signals blended into BI dashboards, each clearly labeled as simulated. |

---

## Key Dependencies

| Feature | Depends on |
|---|---|
| F-05, F-06 | F-03/F-04 grounding (nothing simulates ungrounded) |
| F-07 | F-06; config per §11.1 |
| F-09 | F-07 convergence |
| F-11 | F-09, F-10, F-12 |
| F-13 | F-05, F-06, F-11 |
| F-16, F-17 | F-03, F-04 |
| F-18 | F-09 + logged outcomes (F-13, F-23 feed it) |
| F-27–F-31 | MiroShark fork extensions behind the adapter |
| F-32, F-38 | F-18 validation corpus |
| F-39–F-41 | F-16 monitoring pipeline |
| F-43 | F-20 public API |
| F-45 | Multi-tenant scale (needs enough workspaces to anonymize) |
| F-46 | F-31 + F-40 |

## Milestone Mapping (PBR §13)

| Milestone | Features |
|---|---|
| M1 — Spine | F-03 (Reddit/YouTube first), adapter contracts |
| M2 — First loop | F-02, F-04, F-05, F-06, F-07, F-09 (v0) |
| M3 — Wow | F-01, F-08, F-10, F-11 (v1), F-12, AUTH-1..4, F-27, F-28 |
| M4 — India + GTM | F-13, F-14, F-15, F-39, F-40, F-41 |
| M5 — Enterprise | F-16, F-17, F-18, F-19, F-20, F-21, AUTH-5..8, F-29, F-42, F-43 |
| M6 — Frontier | F-30–F-38, F-44, F-45, F-46 |

*End of features v0.1.*
