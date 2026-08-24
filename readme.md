# Product Blueprint & Requirements (PBR)

# CrowdLens — Crowd Intelligence & Decision Simulation Platform

**Version:** 0.1 (Draft) · **Date:** 2026-08-25 · **Status:** For review

> **One-liner:** CrowdLens tells a business what the world thinks *now* (real, cited data) and simulates what the world will think *next* (grounded multi-agent simulation) — so decisions are tested before money is spent.

---

## 1. Vision & Positioning

Traditional market research is slow (weeks), expensive ($10K–$100K), and static. Social listening is real but backward-looking. CrowdLens fuses both and adds the missing layer: **counterfactual simulation of crowd reaction** to a decision *before* it is made.

**Category:** Business Intelligence → "Decision Intelligence" / Synthetic Market Research
**Tagline:** *Test any decision against a simulated market built from real opinion data.*

**Positioning rules (honesty as strategy):**
- We predict **direction and themes** of crowd reaction, not precise magnitudes.
- Every claim in every output is traceable to a real citation or a labeled simulation run.
- We never sell election-result prediction or trading signals.
- **Domain-agnostic by design** — CrowdLens is a general decision-testing layer, not an ad-testing tool. Advertising (F-13) is one vertical module among many; the same ground → simulate → verdict loop serves any decision with a public-reaction surface.

---

## 2. Product Principles

1. **Grounded or nothing** — no simulation runs without a real-data grounding layer.
2. **Evidence-first UX** — click any claim, see the receipts (real post or simulated quote).
3. **Ensemble by default** — a single simulation run is an anecdote; verdicts require convergence across runs.
4. **Confidence, not theater** — the product visibly admits uncertainty; no fake precision (no invented CTRs, vote shares, revenue numbers).
5. **Global by design, India-native in depth** — multilingual data, tier-2/3 India coverage as a differentiator.

### 2.1 Grounding minimums

"Grounded or nothing" needs teeth. A project may not enter simulation until its grounding corpus clears hard minimums. Defaults below; vertical templates (F-22) may raise them, never lower them.

| Minimum | Default | Checked |
|---|---|---|
| Items per active source | ≥ 100 qualifying items per platform (post-dedup, post-language-filter) | Per collector, at grounding completion |
| Sources per project | ≥ 2 independent platforms contributing qualifying items | Grounding Console |
| Language coverage | Every declared audience language has ≥ 1 contributing source; otherwise the gap is disclosed on the verdict report | Grounding Console + report header |
| Freshness | Corpus covers the project's declared time window; stale-window re-runs require re-grounding | F-16/F-17 monitoring |

**INSUFFICIENT_GROUNDING flow.** If minimums are not met, the project enters the `INSUFFICIENT_GROUNDING` state: simulation and branching endpoints refuse to run, and the UI offers exactly four paths — (1) extend the time window, (2) add sources, (3) upload client data (private-data fusion, below), or (4) stop at the baseline sentiment report (F-04), which is always delivered with a coverage-gap disclosure. There is no "run anyway" button. This is the product principle made mechanical, not a warning banner.

**Niche-B2B policy (private-data fusion path).** Some decisions have a real but thin public surface (industrial components, enterprise software, regulated verticals). For these, the minimums can be met by fusing client-uploaded private data — CRM notes, support tickets, survey responses (F-23), WhatsApp Business exports — with whatever public data exists. Rules: the report header discloses the public/private source mix; private items are citable inside the owning workspace only and never appear in shared/exported reports without explicit per-item release; private uploads inherit the workspace's retention and deletion rules (§11.2). If even fused data can't reach minimums, the honest answer is a baseline report plus a recommendation to run primary research — CrowdLens does not simulate on air.

---

## 3. Target Personas

| Persona | Need | Tier |
|---|---|---|
| **Solo Consultant / Creator** (course launches, webinars, local business) | Cheap, guided, single-project flows | Starter |
| **Agency Strategist** (PR/marketing/digital agencies) | Multi-client workspaces, white-label reports, ad testing | Agency |
| **Enterprise Comms / Insights Lead** | Pre-publication testing, compliance, on-prem option, SSO | Enterprise |
| **Product Manager / Designer** | Concept & feature reaction testing, aspect-level feedback | Agency/Enterprise |
| **Campaign / Policy Analyst** | Issue mining, message testing (within legal guardrails) | Enterprise |

---

## 4. System Architecture

### 4.1 Topology

```
┌──────────────────────────────────────────────────────────────┐
│                     REACT FRONTEND (new)                      │
│   Workspace · Projects · Live Sim View · Report Studio ·      │
│   Knowledge Graph Explorer · Dashboards · Admin               │
└───────────────────────────┬──────────────────────────────────┘
                            │ REST + WebSocket
┌───────────────────────────▼──────────────────────────────────┐
│              CROWDLENS CORE API (our code, closed)            │
│  Auth (Logto) · Projects · Orchestrator · Persona Panel Svc · │
│  Handoff Transformer · Verdict Engine · Report Builder Svc ·  │
│  Billing Meter (Lago) · Notifications (Novu) · Jobs (Temporal)│
└───────┬──────────────────┬───────────────────┬───────────────┘
        │ adapter          │ adapter           │ adapter
┌───────▼───────┐  ┌───────▼────────┐  ┌───────▼────────┐
│  BETTAfish     │  │  MIROSHARK     │  │  RESEARCH SVC  │
│  (Flask svc)   │  │  (Flask svc)   │  │ (GPT-Researcher│
│  collectors →  │  │  simulation    │  │  Apache-2.0,   │
│  MySQL, agents │  │  engine, Neo4j │  │  optional)     │
└───────┬───────┘  └───────┬────────┘  └───────┬────────┘
        └──────────────────┼───────────────────┘
                  ┌────────▼────────┐
                  │  LiteLLM Gateway │  ← one endpoint for all LLM calls:
                  │  routing, cache, │    swarm-model / report-model,
                  │  budgets, keys   │    per-project virtual keys
                  └────────┬────────┘
              DeepSeek / OpenRouter / local vLLM-Ollama (on-prem)
                  
        Langfuse (traces + eval datasets) · Redis (cache/queue)
        PostgreSQL (core data) · MinIO/S3 (artifacts)
```

### 4.2 Service Roles

| Component | Role | Source |
|---|---|---|
| React frontend | Entire user experience (new build) | Ours |
| Core API | Orchestration, projects, verdicts, reports, billing | Ours (closed) |
| BettaFish fork | Real-data collection & opinion analysis | AGPL fork, internal service |
| MiroShark fork | Grounded crowd simulation | AGPL fork, internal service |
| Research service | Open-web/factual research (replaces BettaFish Query Agent) | GPT-Researcher (Apache-2.0) |
| LiteLLM | LLM gateway: model routing, response caching, spend caps, virtual keys, cost ledger | OSS |
| Langfuse | LLM tracing, eval datasets, validation scoring | OSS |
| Logto | Authentication, SSO, multi-tenant orgs | OSS |
| Lago | Metered/credit billing | OSS |
| Novu | Email/Slack/WhatsApp notifications | OSS |
| Temporal | Job orchestration (long-running sims, retries, checkpointing) | OSS |

### 4.3 Data Handoff Contract (the crown jewel)

```
BettaFish output (baseline report + raw comment DB)
        │  Handoff Transformer (our code)
        ▼
MiroShark inputs:
  1. Reality-seed document (MD): facts, sentiment summary, verbatim quotes, key entities
  2. Persona panel: archetypes with proportions, stances, real language samples
  3. Scenario variants: the decision options to test (counterfactual branches)
        │
        ▼
MiroShark output (runs, timelines, agent states, entity graph)
        │  Verdict Engine (our code)
        ▼
CrowdLens Verdict Report (convergence-scored, evidence-linked)
```

---

## 5. Authentication & Accounts

**Provider:** Logto (open-source, self-hostable) behind Core API.

| Req ID | Requirement | Priority |
|---|---|---|
| AUTH-1 | Email + password, magic link, Google/GitHub social login | P0 |
| AUTH-2 | Multi-tenant **Workspaces** (org) with member invites | P0 |
| AUTH-3 | RBAC: Owner / Analyst / Viewer roles per workspace | P0 |
| AUTH-4 | Per-project virtual LLM keys issued via LiteLLM; spend caps per project and per workspace | P0 |
| AUTH-5 | SSO (SAML/OIDC) for Enterprise | P1 |
| AUTH-6 | Personal API keys + scoped service tokens for the public API | P1 |
| AUTH-7 | Audit log of all data access and report exports (Enterprise) | P1 |
| AUTH-8 | On-prem deployments: license-key activation, offline token refresh window | P1 |

---

## 6. Frontend (New React Application)

**Goal:** the best-looking product in the decision-intelligence category. Dark-first, data-dense, cinematic without being slow.

### 6.1 Design language

- **Theme:** dark-first design system; deep-slate background, electric-lime/cyan accent pair; light mode supported
- **Motion:** physics-based transitions (framer-motion); the simulation view *feels alive*
- **Typography:** strong editorial hierarchy; verdicts read like a magazine, not a dashboard
- **Data-viz-first:** every screen leads with a visual (graph, curve, cluster map), numbers second
- **Signature moments:**
  - Live simulation theater: agents post/argue in a real-time animated feed
  - Knowledge graph with time-scrubber and sentiment heat coloring
  - Verdict reveal: one-page decision summary with confidence meter
- Stack: React 18 + TypeScript, Vite, Tailwind + shadcn/ui, framer-motion, react-force-graph/NVL (graph), Recharts/ECharts (charts)

### 6.2 Core screens

1. **Home / Workspaces** — projects as cards with status, cost, confidence
2. **Project Wizard** — question → audience → data sources → variants → budget cap (≤5 steps)
3. **Grounding Console** — live collector status, items per platform, sample quotes, coverage map (incl. India region/language filters)
4. **Persona Panel Editor** — archetype cards, proportion sliders, real-language samples, edit/regenerate
5. **Simulation Theater** — live feed, round scrubber, per-archetype sentiment lanes, branch comparison tabs
6. **Knowledge Graph Explorer** — search, focus mode, evidence drawer, before/after toggle, time scrubber
7. **Report Studio** — interactive report builder (see §8)
8. **Dashboards & Monitoring** — KPI trends, alerts, competitor watchlists
9. **Validation Center** — prediction-vs-reality log, accuracy page, backtest library
10. **Admin** — keys, budgets, members, audit log, deployment profile (cloud/on-prem)

---

## 7. Core Feature Set

**The platform is scenario-universal.** Every domain below runs the identical pipeline — grounding collectors → persona panel → counterfactual simulation → verdict — with only templates, sources, and guardrails varying per vertical. No feature is advertising-specific except the F-13 module itself.

| Domain | Example decisions under test |
|---|---|
| Marketing & advertising | Creative variants, campaign messaging, brand repositioning |
| Product & design | Concept reaction, feature prioritization, naming, packaging |
| Pricing | Price-point reaction, tier restructuring, discount strategy |
| PR & crisis | Statement drafts, apology strategies, crisis response timing |
| Policy & advocacy | Message testing, issue framing (within legal guardrails, §9) |
| Content & media | Show/video concepts, creator strategy, format changes |
| Hiring & employer brand | EVP messaging, layoff/reorg comms reaction |
| Investor & corporate comms | Earnings narrative, announcement framing, M&A messaging |
| Local business | Menu changes, location openings, service pivots |
| Personal / self-serve | Dating profiles, career moves, creator positioning (F-26 funnel) |

Vertical templates (F-22) package per-domain sources, persona packs, and report layouts — the engine underneath never changes.

### P0 — MVP

| ID | Feature | Notes |
|---|---|---|
| F-01 | Projects & workspaces | Multi-client separation for agencies |
| F-02 | Seed document upload (PDF/MD/TXT) | The artifact under test |
| F-03 | Grounding collectors: **Reddit, YouTube, X (provider), Google News/GDELT, Telegram public channels** | Official APIs first |
| F-04 | Baseline sentiment report with citations | Every claim → real source |
| F-05 | Auto persona panel (editable) | Proportions from real data |
| F-06 | Simulation with **counterfactual variant branching** (MiroShark `/branch-counterfactual`) | Up to 5 variants |
| F-07 | Ensemble runs + convergence scoring | Adaptive ensemble, ≥80% agreement to verdict (§11.1) |
| F-08 | Cost estimate + hard spend cap per project | US-3.5 |
| F-09 | Verdict report v1 (ranked options, objections w/ quotes, risk flags, confidence) | One page, executive-readable |
| F-10 | Live simulation view + agent interview (`/ask`) | |

### P1 — Productization

| ID | Feature | Notes |
|---|---|---|
| F-11 | **Interactive Report Studio** (§8) | Blocks, live charts, graph embeds, share links, white-label PDF/PPT export |
| F-12 | Knowledge Graph Explorer | Neo4j `/entities` data + our evidence drawer |
| F-13 | **Ad Testing module** | One vertical module of many (§7 preamble). 10 variants, per-segment verdicts, objection quotes, live-test result upload for calibration |
| F-14 | India data pack (§9) | YouTube India, Reddit India, Telegram, app-store reviews, e-commerce reviews via providers |
| F-15 | Multilingual analysis | Marathi/Hindi/Hinglish content; reports in EN + HI/MR |
| F-16 | Continuous monitoring & alerts | Scheduled collectors, sentiment-shift alerts (WhatsApp/email) |
| F-17 | Scheduled re-simulation | "Re-run with last week's data" subscriptions |
| F-18 | Validation corpus + public accuracy page | Langfuse datasets; the enterprise sales asset |
| F-19 | White-label agency exports | Client-branded PDF/PPT |
| F-20 | Public API + webhooks | For agency/enterprise embedding |
| F-27 | Agent memory & relationship graph | Agents remember interactions across rounds; influence accumulates through persistent social ties, not per-round resets |
| F-28 | Seeded influencer agents | High-reach voices detected in grounding data become high-centrality agents with their real style and stance |
| F-29 | Mid-simulation event injection | Scripted shocks at chosen rounds (viral post, price change, news event) — crisis and resilience testing |
| F-39 | Narrative tracking & momentum | Posts clustered into narratives; each tracked through its lifecycle (emerging → peaking → declining) with a momentum score |
| F-40 | Anomaly detection & crisis early-warning | Statistical spike detection on all KPIs; composite crisis score with driver drill-down to the posts causing it |
| F-41 | Automated insight digests | Scheduled plain-language briefings per project: what changed, why, what to watch |

### P2 — Expansion

| ID | Feature | Notes |
|---|---|---|
| F-21 | On-prem / air-gapped deployment profile (local models via vLLM/Ollama; license keys) | |
| F-22 | Vertical template marketplace (PR crisis, course launch, product design, dating, campaign intelligence, restaurant/local business, YouTube growth) | |
| F-23 | Client ground-truth upload (surveys via Formbricks, recordings via Whisper) → auto-validation | |
| F-24 | Aspect-based sentiment layer (PyABSA) in verdicts — "loved the price, hated the name" | |
| F-25 | Collaboration: comments on reports, shared workspaces, approval flows | |
| F-26 | Dating-profile/consumer self-serve mini-tool as viral funnel | |
| F-30 | Adversarial stress-testing | Devil's-advocate agent swarms attack the leading option; a verdict must survive the challenge round to ship |
| F-31 | Longitudinal simulation | Multi-week runs with scheduled re-grounding; per-variant sentiment drift curves over time |
| F-32 | Synthetic-survey calibration | Agent opinion distributions benchmarked against real poll/survey data before a run is accepted |
| F-42 | Custom KPI builder | Calculated metrics and workspace-composed dashboards on top of grounding + simulation data |
| F-43 | BI connectors | Native Looker / Tableau / Power BI / Google Sheets connectors atop the public API (F-20) |
| F-44 | Segment & cohort analytics | Cuts by region, language, and persona archetype; cohort trend comparison over time |
| F-45 | Benchmark indices | Opt-in, anonymized cross-tenant benchmarks — share-of-voice and sentiment norms per vertical |

### P3 — Frontier (Simulation & AI Depth)

Longer-horizon bets that deepen the simulation core. All remain bound by the Product Principles — grounded, ensemble, honest-confidence.

| ID | Feature | Notes |
|---|---|---|
| F-33 | Counterfactual tree search | Engine proposes the decision space itself: auto-generates variant branches, explores, and prunes low-signal ones within budget |
| F-34 | Narrative trajectory forecasting | Theme-emergence prediction over 2–8 week horizons; every forecast logged and scored against the validation corpus |
| F-35 | Archetype-distilled models | Fine-tune small models per persona archetype to cut ensemble run cost without changing agent distributions |
| F-36 | Generative focus groups | Live moderated sessions with an agent subset; the analyst asks follow-ups in real time, answers carry sim-run labels |
| F-37 | Multi-public simulation | Linked simulations across regions/languages; narratives migrate between publics (e.g. English-urban → Hindi tier-2/3) |
| F-38 | Auto-calibration loop | Logged verdict outcomes continuously tune persona proportions; human-approved updates, full drift audit trail |
| F-46 | Simulated leading indicators | Simulation-derived early-warning signals blended into BI dashboards — "the swarm turned before the real crowd did" |

---

## 8. Interactive Report Generation (Flagship Feature)

Static PDFs are where insight goes to die. CrowdLens reports are **living documents**.

### 8.1 Report model

A report = ordered **blocks**, each live and evidence-linked:

| Block type | Behavior |
|---|---|
| Verdict card | Recommendation + confidence meter + convergence score |
| Sentiment timeline | Real-data trend with event annotations |
| Simulation comparison | Variant ranking chart; click a bar → that branch's theater replay |
| Persona panel | Archetype cards with stance meters and real quote samples |
| Knowledge graph embed | Fully interactive, embedded in the report |
| Objection map | Clustered objections, each expandable to real/simulated quotes |
| Evidence rail | Every figure hoverable → source post (platform, date, link) or labeled sim run |
| Cost ledger | What this report cost to produce (builds trust + agency markup transparency) |

### 8.2 Interactions

- **Ask-the-report chat** — ask follow-ups; answers cite blocks and sources (RAG over project data)
- **Branch what-ifs** — "rerun this section with price ₹399" directly from the report (spawns a new counterfactual)
- **Live mode** — monitoring-enabled reports update on schedule; diffs highlighted ("sentiment dropped 12 pts since Tuesday — here's the thread driving it")
- **Share links** — view-only, expiring, watermarked; per-link analytics for agencies
- **Exports** — white-label PDF and PPTX (client logo, colors), plus raw data (CSV/JSON)

### 8.3 Generation pipeline

Report Builder service (ours) composes blocks from: BettaFish analysis artifacts + MiroShark run data + verdict engine scores → templates per vertical → Langfuse-evaluated for claim-evidence consistency before publish.

---

## 9. Data Source Matrix (Global + India)

### 9.1 Global core

| Source | Access | Status |
|---|---|---|
| Reddit | Official API (PRAW) | ✅ P0 |
| YouTube | Data API v3 (comments, search, captions) | ✅ P0 |
| X/Twitter | Official pay-per-read or licensed provider | ✅ P0 |
| News/web | GDELT, RSS, Google News, Research service | ✅ P0 |
| Telegram | **Open API — public channels/groups** (huge in India) | ✅ P1 |

### 9.2 India pack

India reality (2026): WhatsApp ~535M, YouTube ~500M, Instagram ~480M, Facebook ~400M, **ShareChat ~350M regional-language MAU**, Moj ~160M, Telegram ~104M, LinkedIn ~170M, Reddit ~31M, X ~22M users. Coverage strategy:

| Source | Access path | Notes |
|---|---|---|
| YouTube India (Hindi/Marathi/regional) | Official API | Highest-signal tier-2/3 source — comment culture is massive |
| Reddit India subs | Official API | English-leaning, urban |
| **Telegram public channels** | Open API | Underrated goldmine: Indian news, deals, local communities |
| Google Play / App Store **app reviews** | Scraper libraries | Enormous for app/BI use cases; near-official |
| Flipkart / Amazon.in product reviews | Third-party review APIs (official APIs don't expose review text) | Product/market intelligence; budget per volume |
| ShareChat / Moj | ❌ No public API — provider-dependent | P2, opportunistic |
| Instagram / Facebook | Own-account official API only; providers for public data | Gray zone; never core dependency |
| WhatsApp | Business API (client's own channels only) | Client-uploaded exports for private-domain fusion |
| LinkedIn | Client's own org page (Marketing API) | B2B add-on only |
| Regional news (Marathi/Hindi portals) | RSS + GDELT + Research service | Local-election/local-business verticals |

**Legal/compliance guardrails baked in:** official/licensed sources by default; scraping only public data within robots/ToS boundaries; India **DPDP Act 2023** + GDPR compliance — aggressive anonymization, no PII storage of platform users, aggregation-only publishing. Election-period publishing restrictions (RPA §126A) enforced as a product rule for campaign verticals.

---

## 10. Business Intelligence Layer

CrowdLens as a **complete BI tool for external perception**:

1. **Dashboards** — brand/topic KPIs: sentiment index, share-of-voice, issue volume, narrative momentum; per-market and per-language cuts
2. **Competitor watchlists** — tracked entities with alerts on movement
3. **Decision registry** — every verdict logged with outcome; org-level accuracy analytics
4. **Exports & embedding** — scheduled report digests, API, iframe-embeddable blocks for client portals
5. **Private data fusion** — client CRM/support/survey data merged with public opinion (BettaFish's private-DB pattern)
6. **Marketplace of analyses** — prebuilt vertical templates and persona panel packs (India architects, US Gen-Z consumers, EU policy publics...)
7. **Narrative tracking** (F-39) — emerging narratives auto-clustered and tracked through their lifecycle with momentum scores; the "what is the crowd actually saying" layer
8. **Anomaly detection & crisis early-warning** (F-40) — spike detection across all KPIs; composite crisis score with drill-down to the specific posts driving it
9. **Automated insight digests** (F-41) — scheduled plain-language briefings: what changed, why, what to watch; delivered via Novu (email/Slack/WhatsApp)
10. **Custom KPI builder** (F-42) — calculated metrics and workspace-composed dashboards, no vendor lock-in on definitions
11. **BI connectors** (F-43) — native Looker / Tableau / Power BI / Google Sheets connectors so CrowdLens feeds existing analyst stacks instead of replacing them
12. **Segment & cohort analytics** (F-44) — region, language, and archetype cuts; cohort trend comparison (e.g. tier-2 Hindi speakers vs metro English)
13. **Benchmark indices** (F-45) — opt-in anonymized cross-tenant norms: "your sentiment is 8 pts below vertical median"
14. **Simulated leading indicators** (F-46) — P3: simulation-derived early-warning signals blended into dashboards, each clearly labeled as simulated per the honesty principles

---

## 11. Non-Functional Requirements

| Area | Requirement |
|---|---|
| Performance | Grounding baseline ≤ 30 min; simulation verdict same-day; graph UI smooth to 5K nodes |
| Cost | Verdict report all-in ≤ $25 at standard config; per-project hard caps enforced by LiteLLM |
| Reliability | Sims checkpoint every N rounds; resume after restart; Temporal-managed retries |
| Security | Encrypted at rest; per-workspace isolation; audit logs; SSO for enterprise |
| Compliance | DPDP (India), GDPR; no PII of platform users persisted; citation-based transparency |
| Deployment | Cloud (SaaS) + on-prem profile (single docker-compose / Helm; GPU sizing tiers 24GB/48GB/2×80GB) |
| Licensing | AGPL forks run as internal services behind adapters; our layer closed; on-prem distribution offers fork source to customers |

### 11.1 Cost-Optimized Standard Configuration

The default config is engineered for cheapest-accurate: the crowd runs on the cheapest model, judgment runs on the strong one.

| Lever | Setting |
|---|---|
| Model routing (LiteLLM) | `swarm-model` = DeepSeek-V4-Flash (all agent turns); `report-model` = DeepSeek-V4-Pro (grounding analysis + verdict engine only) |
| Prompt caching | Persona cards, reality-seed doc, system prompts cached; repeated input billed at cache-hit rate (~50× cheaper) |
| Ensemble | Adaptive: start 3 runs, add until ≥80% agreement, cap 7 (replaces fixed-5; never below 3) |
| Simulation scale | 120 agents × 18 rounds, early-stop when sentiment delta < threshold for 3 consecutive rounds |
| Variants | ≤3 counterfactual branches at MVP |
| Guardrail | Grounding budget is untouchable — cost cuts never come out of collector volume or analysis quality |

Expected simulation-layer cost at this config: **~$2–3 per decision cycle** (single run ~$0.10–0.15), leaving most of the $25 budget for grounding and report generation. Any further cuts require validation-corpus evidence (F-18) that theme-level accuracy holds.

### 11.2 Data Lifecycle & Retention

Retention is per data class, enforced by scheduled jobs (Temporal), and auditable. Rule of thumb: **keep the evidence, expire the exhaust** — raw text ages out; derived aggregates and verdicts live with the project.

| Data class | Where | Retention | On project delete |
|---|---|---|---|
| Raw collected posts (text) | PostgreSQL `collected_items` | 90 days from collection, then text purged | Hard-deleted immediately |
| Aggregates, embeddings, entity stats | PostgreSQL / Neo4j | Project lifetime | Hard-deleted with project graph |
| Persona panels (derived, anonymized) | PostgreSQL | Project lifetime | Hard-deleted |
| Simulation artifacts (runs, agent states, timelines) | MinIO + PostgreSQL | Project lifetime | Hard-deleted |
| Reports & report blocks | PostgreSQL + MinIO exports | Project lifetime; share links expire ≤ 30 days (enforced at creation) | Hard-deleted; share links revoked |
| Client-uploaded private data (§2.1) | PostgreSQL + MinIO | Project lifetime, or earlier per client instruction | Hard-deleted first, before public data |
| Audit logs (access, exports) | PostgreSQL | 12 months minimum (Enterprise); retained post-churn in workspace-anonymized form | Survives churn, anonymized |
| LLM traces (Langfuse) | Langfuse | 90 days, then purged; no raw PII enters traces | Purged with workspace |
| Backups | Encrypted snapshots | 30-day rotation | Data ages out of backups within 30 days; no restore-on-demand for churned workspaces |

**Deletion-on-churn flow.** Workspace closure triggers: (1) 30-day grace period — read-only, full export available, no new jobs; (2) hard delete — cascade through PostgreSQL (workspace → projects → all child rows, per DDL), purge MinIO buckets, drop Neo4j project graphs, revoke LiteLLM virtual keys, delete Lago/Novu tenant config; (3) backups rotate out within 30 days; (4) Enterprise customers receive a signed deletion certificate listing what was destroyed and what survives (anonymized audit logs only). No soft-delete shadow copies.

**DPDP / GDPR mapping.**

| Obligation | How CrowdLens meets it |
|---|---|
| No platform-user PII | Handles, avatars, profile links stripped at ingestion (verified in collector tests); platform users are never data principals in our stores |
| Data minimization | Per-class retention above; raw text expires, only aggregates persist |
| Purpose limitation | Client data used only for the project it was uploaded to; no cross-tenant reuse (F-45 benchmarks are opt-in and aggregate-only) |
| Right to erasure | Churn flow above; client instructs → we delete, certificate issued |
| Consent / legitimate use | Public posts processed under legitimate-interest (GDPR) / publicly-available-data provisions; private uploads under the client's own lawful basis |
| DPDP specifics | Grievance officer designated; breach notification to Data Protection Board + affected clients; no transfer to GoI-blacklisted jurisdictions |
| GDPR specifics | DPA offered to EU clients; EU-region storage pinning available on Enterprise; SCCs for any cross-border processing |

---

## 12. User Story Traceability

Epics 1–7 from the story set map to features: Project Setup (F-01/02), Grounding (F-03/04/05), Simulation (F-06/07/08/10, extended by F-27–F-38), Decision Output (F-09/11/12/19), Ad Testing (F-13), Validation (F-18/23, extended by F-32/34/38), Platform/Admin (AUTH-*, F-16/17/20/21). The BI layer (F-39–F-46) builds on Grounding + Decision Output and extends Epic 4 and Epic 7 stories.

**MVP cut:** F-01–F-10 + AUTH-1..4 — "real data in → verdict out, with cost control."

---

## 13. Roadmap

| Milestone | Scope | Target |
|---|---|---|
| **M1 — Spine** (wks 1–3) | Clone forks; adapter contracts; LiteLLM + Langfuse up; Reddit/YouTube collectors into BettaFish DB | |
| **M2 — First loop** (wks 4–6) | Handoff transformer; MiroShark branching runs; verdict engine v0; minimal React shell (wizard + verdict page) | |
| **M3 — Wow** (wks 7–10) | Simulation theater, knowledge graph explorer, Report Studio v1, auth/workspaces, Lago billing, agent memory + seeded influencers (F-27/28) | |
| **M4 — India + GTM** (wks 11–14) | India data pack, multilingual, ad-testing module, narrative tracking + anomaly alerts + digests (F-39–F-41), 5 agency pilots, first backtests published | |
| **M5 — Enterprise** (wks 15–20) | Monitoring/alerts, public API, on-prem profile, SSO, accuracy page, shock injection (F-29), KPI builder + BI connectors (F-42/43) | |
| **M6 — Frontier** (wks 21+) | P3 simulation depth: counterfactual tree search, trajectory forecasting, distilled archetype models, multi-public sims; adversarial + longitudinal sims (F-30/31), calibration (F-32), segment analytics + benchmarks (F-44/45), simulated leading indicators (F-46) | |

---

## 14. Pricing Model (Credit-based, Lago-metered)

| Tier | Price | Includes |
|---|---|---|
| Starter | ₹4,999 / $59 mo | 10 credits, self-serve, community panels |
| Agency | ₹24,999 / $299 mo | 60 credits, white-label, 5 seats, ad testing |
| Enterprise | Custom ($5–15K mo) | SSO, on-prem option, private panels, SLA, accuracy reports |

1 credit ≈ one full decision cycle (grounding + branched ensemble simulation + verdict).

---

## 15. Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Simulation trust | Validation corpus, public accuracy page, convergence requirements, honest-confidence UI |
| AGPL obligations | Adapter isolation; open-core decision pre-launch; source offered to on-prem customers |
| Upstream maintainer drift (MiroShark) | Fork discipline; thin patches; engine swappable behind adapter |
| India data gaps (WhatsApp dark, ShareChat no API) | Disclosed coverage maps per report; client-data fusion; never oversell |
| Platform ToS shifts | Official/licensed sources first; provider abstraction per collector |
| Legal (elections, defamation) | Vertical guardrails in product; §126A publishing lock; advisory-only positioning |

---

## Competitive Landscape

*(Inserted in the v0.1 draft between §15 and §16. Section numbering is frozen for this draft — existing §16 stays §16; this section takes a number at the next version bump.)*

Three adjacent categories, none of which does the full ground → simulate → verdict loop:

| Category (examples) | What it does | Where it stops | What CrowdLens does differently | Honest CrowdLens weaknesses |
|---|---|---|---|---|
| **Synthetic-audience / AI-persona tools** (synthetic-respondent startups, LLM "digital twin" panels) | Ask LLM-generated personas for reactions; instant, cheap | Personas come from model priors, not live cited data; no proof the panel matches a real public; answers unverifiable | Persona panel is built from real, cited posts (F-05); every agent stance traceable to grounding data; ensemble + convergence before any verdict (§11.1) | Slower (grounding ≤ 30 min before anything runs) and dearer than a pure-LLM chat; accuracy still being proven against the validation corpus (F-18) |
| **Social listening incumbents** (Brandwatch, Sprinklr, Talkwalker, Meltwater) | Real-time monitoring of public conversation, dashboards, alerts | Backward-looking: tells you what the crowd said, never what it *will* say to a decision you haven't made | Same real-data spine, plus counterfactual simulation of decision variants (F-06) and convergence-scored verdicts (F-09) | Their data volume, historical archives, and brand trust dwarf ours; our coverage has disclosed gaps (WhatsApp dark, ShareChat no API — §9.2) |
| **Traditional research** (Nielsen/Kantar panels, Qualtrics surveys, focus groups) | Verified-demographic human respondents; the methodological gold standard | Weeks and $10K–$100K per study; static snapshot; can't iterate 5 variants overnight | Same-day verdict at ≤ $25 (§16); decisions testable before money is spent; verdicts re-runnable as data refreshes (F-17) | No verified demographics — we predict themes and direction, not statistically representative magnitudes (§1); we are a complement to, not a replacement for, a real panel on high-stakes calls |

**Positioning in one line:** the incumbents own *what happened*, the synthetic tools sell *unverified what's next* — CrowdLens sells *grounded what's next*, and publishes its accuracy to prove it (F-18).

---

## 16. Success Metrics

- Verdict accuracy (prediction-vs-reality hit rate) — target ≥70% theme-level convergence on logged outcomes
- Time-to-verdict < 24h; cost-per-verdict < $25
- Weekly active projects per agency seat; credit burn rate; expansion revenue
- On-prem deployments closed in first 2 quarters ≥ 3

---

*End of PBR v0.1 — companion artifacts: user-stories.md (story set), features.md (feature catalog), ai-dev/ (build kit: contracts, phased instructions, manual tests). Next artifacts: UX wireframes, accuracy methodology spec, GTM plan.*
