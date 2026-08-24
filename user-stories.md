# CrowdLens — User Stories

**Version:** 0.1 (Draft) · **Date:** 2026-08-25 · **Companion to:** readme.md (PBR) v0.1

Stories are grouped into 7 epics. Every story maps to feature IDs in PBR §7 and auth requirements in PBR §5. Story IDs follow `US-<epic>.<n>`.

**Personas** (PBR §3): Solo Consultant (SC), Agency Strategist (AS), Enterprise Comms Lead (EC), Product Manager (PM), Campaign/Policy Analyst (CA).

**Global acceptance rule:** any story whose output makes a claim must satisfy the Product Principles — every claim traceable to a real citation or a labeled simulation run, verdicts only on ensemble convergence, uncertainty shown honestly.

---

## Epic 1 — Project Setup (F-01, F-02)

### US-1.1 — Create a project in a workspace
**As an** Agency Strategist, **I want** to create projects inside separate client workspaces, **so that** each client's data, keys, and budgets stay isolated.

**Acceptance criteria**
- Project created under a workspace inherits its members, roles, and spend caps.
- Project card shows status, accrued cost, and confidence (when available).
- Viewer-role members cannot create or edit projects.
- **Maps to:** F-01, AUTH-2, AUTH-3

### US-1.2 — Guided project wizard
**As a** Solo Consultant, **I want** a wizard that walks me from question to budget cap in ≤5 steps, **so that** I can launch a decision test without training.

**Acceptance criteria**
- Steps: decision question → audience → data sources → variants → budget cap.
- Each step has sane defaults; a project is launchable with only the question filled in.
- **Maps to:** F-01, F-08

### US-1.3 — Upload the seed artifact
**As a** Product Manager, **I want** to upload the artifact under test (PDF/MD/TXT — concept note, landing page, ad copy), **so that** the simulation reacts to the real thing, not a paraphrase.

**Acceptance criteria**
- PDF/MD/TXT accepted; text extracted and previewed before launch.
- The uploaded artifact is stored per-project and versioned on re-upload.
- **Maps to:** F-02

---

## Epic 2 — Grounding (F-03, F-04, F-05, F-14, F-15)

### US-2.1 — Collect real opinion data
**As an** Analyst, **I want** the platform to collect real posts from Reddit, YouTube, X, news/GDELT, and Telegram about my topic, **so that** every downstream claim is grounded in reality.

**Acceptance criteria**
- Collectors run per official/licensed APIs; per-source status visible in the Grounding Console.
- Coverage gaps are disclosed (e.g. "no ShareChat access") — never silently omitted.
- **Maps to:** F-03

### US-2.2 — Baseline sentiment report with receipts
**As an** Enterprise Comms Lead, **I want** a baseline report where every claim links to the real post it came from, **so that** I can defend the analysis in front of leadership.

**Acceptance criteria**
- Every claim carries ≥1 citation: platform, date, link.
- Claims with insufficient evidence are flagged, not smoothed over.
- **Maps to:** F-04

### US-2.3 — Auto-generated persona panel
**As a** Solo Consultant, **I want** the system to propose audience archetypes with proportions derived from the real data, **so that** the simulated crowd looks like my actual market.

**Acceptance criteria**
- Archetypes include proportions, stances, and real language samples.
- Panel is editable (rename, reweight, remove) before simulation.
- **Maps to:** F-05

### US-2.4 — India-native coverage
**As an** Agency Strategist serving Indian clients, **I want** Hindi/Marathi/regional sources (YouTube India, Telegram, app-store and e-commerce reviews), **so that** tier-2/3 opinion is represented.

**Acceptance criteria**
- India pack sources selectable per project; region/language filters in the Grounding Console.
- Analysis handles Hindi/Marathi/Hinglish content; reports render in EN + HI/MR.
- **Maps to:** F-14, F-15

---

## Epic 3 — Simulation (F-06–F-08, F-10, F-27–F-31)

### US-3.1 — Counterfactual variant branching
**As a** Product Manager, **I want** to test up to 3 decision variants (e.g. price ₹299 vs ₹399 vs ₹499) as parallel simulation branches, **so that** I can compare reactions before committing.

**Acceptance criteria**
- Each variant spawns an independent branch from the same grounded seed.
- Branch results are comparable side-by-side in the Simulation Theater.
- **Maps to:** F-06 (MVP cap: 3 variants per §11.1)

### US-3.2 — Watch the crowd react live
**As an** Agency Strategist, **I want** a live feed of agents posting and arguing per round, **so that** I can see *how* opinion forms, not just the final number.

**Acceptance criteria**
- Real-time feed with round scrubber and per-archetype sentiment lanes.
- Agents remember prior interactions (memory + relationship graph).
- **Maps to:** F-10, F-27

### US-3.3 — Interview an agent
**As a** Campaign Analyst, **I want** to ask an individual agent why it reacted the way it did, **so that** I can surface objections that aggregates hide.

**Acceptance criteria**
- `/ask` returns answers labeled as simulated, citing the agent's run and round.
- **Maps to:** F-10

### US-3.4 — Ensemble convergence before verdict
**As an** Enterprise Comms Lead, **I want** a verdict to be issued only when independent runs converge, **so that** I'm not acting on a single lucky simulation.

**Acceptance criteria**
- Adaptive ensemble: starts at 3 runs, extends until ≥80% agreement, caps at 7.
- Non-converging decisions return "no verdict" with divergence analysis — never a forced answer.
- **Maps to:** F-07 (config per §11.1)

### US-3.5 — Cost estimate and hard spend cap
**As a** Solo Consultant, **I want** an upfront cost estimate and a hard cap per project, **so that** a runaway simulation can never surprise my invoice.

**Acceptance criteria**
- Estimate shown before launch; project halts (not just warns) at cap.
- Caps enforced at LiteLLM via per-project virtual keys.
- **Maps to:** F-08, AUTH-4

### US-3.6 — Mid-simulation shock injection
**As an** Enterprise Comms Lead, **I want** to inject an event at a chosen round ("a negative review goes viral at round 10"), **so that** I can test my crisis response, not just the launch.

**Acceptance criteria**
- Scripted events injectable per branch; effect visible in sentiment lanes.
- **Maps to:** F-29

### US-3.7 — Adversarial stress test
**As a** Product Manager, **I want** a devil's-advocate swarm to attack the leading option, **so that** the winning variant survives genuine opposition.

**Acceptance criteria**
- Challenge round runs per verdict; verdict report includes the strongest attack and its rebuttal.
- **Maps to:** F-30

### US-3.8 — Seeded influencer agents
**As an** Agency Strategist, **I want** the real high-reach voices found in grounding data to appear in the simulation as influential agents with their actual style and stance, **so that** the crowd reacts to amplified voices the way the real market would.

**Acceptance criteria**
- Influencers are detected from grounding metrics (reach, centrality), never invented.
- Influencer agents are labeled in the theater view; their stance traces to real posts.
- **Maps to:** F-28

### US-3.9 — Longitudinal simulation
**As an** Enterprise Comms Lead, **I want** multi-week simulations with scheduled re-grounding, **so that** I can watch sentiment drift per variant instead of betting on a single snapshot.

**Acceptance criteria**
- Runs re-ground on schedule; per-variant drift curves comparable over time.
- Drift curves labeled as simulated; each data point links to its run.
- **Maps to:** F-31

---

## Epic 4 — Decision Output (F-09, F-11, F-12, F-19, F-39–F-42, F-44)

### US-4.1 — One-page verdict
**As an** Enterprise Comms Lead, **I want** a one-page verdict — ranked options, top objections with quotes, risk flags, and a confidence meter — **so that** a decision-maker gets the answer in 60 seconds.

**Acceptance criteria**
- Confidence and convergence scores visible; no invented magnitudes (no fake CTRs or vote shares).
- Every objection expandable to real or labeled-simulated quotes.
- **Maps to:** F-09

### US-4.2 — Living report
**As an** Agency Strategist, **I want** interactive report blocks (timelines, graphs, objection maps) with an evidence rail, **so that** clients explore the receipts instead of reading a dead PDF.

**Acceptance criteria**
- Blocks per PBR §8.1; every figure hoverable to its source.
- Ask-the-report chat answers cite blocks and sources.
- **Maps to:** F-11

### US-4.3 — Branch what-ifs from the report
**As a** Product Manager, **I want** to rerun a section with a tweaked parameter ("what if ₹399?") directly from the report, **so that** follow-up questions don't require a new project.

**Acceptance criteria**
- Spawns a new counterfactual branch inheriting the project's grounding.
- New branch runs under the same spend cap and ensemble rules.
- **Maps to:** F-11, F-06

### US-4.4 — Knowledge graph exploration
**As an** Analyst, **I want** to explore the entity graph with a time scrubber and before/after toggle, **so that** I can trace which narratives moved opinion.

**Acceptance criteria**
- Search, focus mode, evidence drawer per entity.
- Smooth to 5K nodes (PBR §11).
- **Maps to:** F-12

### US-4.5 — White-label export
**As an** Agency Strategist, **I want** client-branded PDF/PPTX exports and expiring share links, **so that** deliverables look like my agency made them.

**Acceptance criteria**
- Client logo/colors applied; share links view-only, expiring, watermarked.
- **Maps to:** F-19

### US-4.6 — Narrative tracking
**As an** Analyst, **I want** posts auto-clustered into named narratives with lifecycle stage (emerging/peaking/declining) and momentum, **so that** I see which story is winning, not just aggregate sentiment.

**Acceptance criteria**
- Narratives tracked across monitoring runs; each expandable to driver items.
- Clusters must pass human eyeball review before a dashboard ships them (no decorative clusters).
- **Maps to:** F-39

### US-4.7 — Anomaly detection & crisis early-warning
**As an** Enterprise Comms Lead, **I want** statistical spike detection with a composite crisis score and drill-down to the posts driving it, **so that** I hear about a brewing crisis before it trends.

**Acceptance criteria**
- Every alert links driver items; an anomaly without drivers never alerts.
- Thresholds documented and configurable per project.
- **Maps to:** F-40

### US-4.8 — Automated insight digests
**As a** Solo Consultant, **I want** a scheduled plain-language briefing — what changed, why, what to watch — **so that** I stay informed without opening dashboards.

**Acceptance criteria**
- Every claim in the digest cited; delivered via email/WhatsApp per notification prefs.
- **Maps to:** F-41

### US-4.9 — Custom KPIs & segment analytics
**As an** Agency Strategist, **I want** calculated metrics, workspace-composed dashboards, and cuts by region/language/archetype, **so that** each client sees the KPIs their business cares about.

**Acceptance criteria**
- Metric definitions stored per workspace; every data point opens the evidence drawer.
- **Maps to:** F-42, F-44

---

## Epic 5 — Ad Testing (F-13)

### US-5.1 — Test creative variants per segment
**As an** Agency Strategist, **I want** to run up to 10 ad variants against persona segments and get per-segment verdicts with objection quotes, **so that** I kill weak creative before paying for media.

**Acceptance criteria**
- Per-segment ranking with objections linked to simulated quotes.
- Honest output: predicted direction and objections only — no invented CTR predictions.
- **Maps to:** F-13

### US-5.2 — Upload live-test results for calibration
**As an** Agency Strategist, **I want** to upload real campaign results after launch, **so that** the system learns where its predictions were right or wrong.

**Acceptance criteria**
- Uploaded outcomes join the validation corpus and the prediction-vs-reality log.
- **Maps to:** F-13, F-18

---

## Epic 6 — Validation (F-18, F-23, F-32, F-34, F-38)

### US-6.1 — Prediction-vs-reality log
**As an** Enterprise Comms Lead, **I want** every verdict logged against its real-world outcome, **so that** I know the platform's actual hit rate before I trust it with a big call.

**Acceptance criteria**
- Decision registry stores verdict, confidence, and eventual outcome.
- Org-level accuracy analytics visible in the Validation Center.
- **Maps to:** F-18

### US-6.2 — Public accuracy page
**As a** prospective customer, **I want** a public page of backtests and logged accuracy, **so that** I can verify claims before buying.

**Acceptance criteria**
- Published backtests with methodology; failures shown alongside hits.
- **Maps to:** F-18

### US-6.3 — Client ground-truth upload
**As an** Enterprise Comms Lead, **I want** to upload our own surveys and interview recordings, **so that** simulations calibrate against *our* customers' reality.

**Acceptance criteria**
- Surveys via Formbricks, recordings transcribed via Whisper; used only for that tenant's calibration.
- **Maps to:** F-23

### US-6.4 — Synthetic-survey calibration gate
**As the** platform, **I want** agent opinion distributions benchmarked against real poll/survey data before a run is accepted, **so that** a drifting persona panel can't silently degrade verdicts.

**Acceptance criteria**
- Failed calibration blocks verdict issuance and flags the panel for regeneration.
- **Maps to:** F-32

---

## Epic 7 — Platform & Admin (AUTH-*, F-16, F-17, F-20, F-21, F-25)

### US-7.1 — Sign in and join a workspace
**As a** new user, **I want** email, magic-link, or Google/GitHub login and workspace invites, **so that** I'm productive in minutes.

**Acceptance criteria**
- AUTH-1/2 flows; invited members land in the right workspace with the right role.
- **Maps to:** AUTH-1, AUTH-2

### US-7.2 — Role-based access
**As a** Workspace Owner, **I want** Owner/Analyst/Viewer roles, **so that** clients can view reports without touching project settings.

**Acceptance criteria**
- Role matrix enforced in API and UI; role changes take effect immediately.
- **Maps to:** AUTH-3

### US-7.3 — Monitoring and alerts
**As an** Enterprise Comms Lead, **I want** scheduled collectors with sentiment-shift alerts to WhatsApp/email, **so that** I hear about a narrative shift before my boss does.

**Acceptance criteria**
- Alert thresholds configurable per project; alert links to the driving thread.
- **Maps to:** F-16

### US-7.4 — Scheduled re-simulation
**As an** Agency Strategist, **I want** "re-run with last week's data" subscriptions, **so that** verdicts stay current as opinion moves.

**Acceptance criteria**
- Scheduled cycles re-ground then re-simulate; report diffs highlighted.
- **Maps to:** F-17

### US-7.5 — Public API and webhooks
**As an** Enterprise engineer, **I want** API keys and webhooks for projects and verdicts, **so that** CrowdLens embeds in our existing BI stack.

**Acceptance criteria**
- Scoped service tokens; webhook events for grounding done, sim done, verdict issued.
- **Maps to:** F-20, AUTH-6

### US-7.6 — On-prem deployment
**As an** Enterprise IT lead, **I want** a single docker-compose/Helm install with local models and license-key activation, **so that** sensitive decision data never leaves our network.

**Acceptance criteria**
- Air-gapped profile per PBR §11; offline token refresh window; fork source offered per AGPL.
- **Maps to:** F-21, AUTH-8

### US-7.7 — Audit log and SSO
**As an** Enterprise compliance officer, **I want** SAML/OIDC SSO and a full audit log of data access and exports, **so that** we pass security review.

**Acceptance criteria**
- SSO per AUTH-5; audit events queryable and exportable per AUTH-7.
- **Maps to:** AUTH-5, AUTH-7

### US-7.8 — BI connectors & benchmarks
**As an** Enterprise insights lead, **I want** native Looker/Tableau/Power BI/Sheets connectors and opt-in anonymized benchmark indices, **so that** CrowdLens feeds our existing stack and we can compare against vertical norms.

**Acceptance criteria**
- Connectors sit on the public API (F-20) — no separate data path.
- Benchmarks strictly opt-in and anonymized; a workspace can withdraw and have its aggregates removed.
- **Maps to:** F-43, F-45

---

## Traceability Summary

| Epic | Stories | Features |
|---|---|---|
| 1. Project Setup | US-1.1–1.3 | F-01, F-02 |
| 2. Grounding | US-2.1–2.4 | F-03, F-04, F-05, F-14, F-15 |
| 3. Simulation | US-3.1–3.9 | F-06, F-07, F-08, F-10, F-27–F-31 |
| 4. Decision Output | US-4.1–4.9 | F-09, F-11, F-12, F-19, F-39–F-42, F-44 |
| 5. Ad Testing | US-5.1–5.2 | F-13, F-18 |
| 6. Validation | US-6.1–6.4 | F-18, F-23, F-32 |
| 7. Platform & Admin | US-7.1–7.8 | AUTH-1..8, F-16, F-17, F-20, F-21, F-43, F-45 |

**MVP cut (PBR §12):** US-1.1–1.3, US-2.1–2.3, US-3.1–3.5, US-4.1, US-7.1–7.2 + spend-cap enforcement.

*End of user-stories v0.1.*
