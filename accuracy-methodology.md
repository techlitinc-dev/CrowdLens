# CrowdLens — Accuracy Methodology

**Version:** 0.1 (Draft) · **Date:** 2026-08-25 · **Companion to:** readme.md (PBR) §2/§16, features.md (F-18, F-23, F-32, F-34), user-stories.md (Epic 6), ai-dev/contracts/adapter-contract.md §3, ai-dev/contracts/data-model.md (`outcome_logs`)

This spec defines how CrowdLens measures whether its predictions were right. It is the scoring contract behind the Validation Center (F-18), the public accuracy page (US-6.2), backtests, and client ground-truth calibration (F-23).

**Positioning rules (PBR §1/§2) this spec enforces:**
- We predict **direction and themes**, never magnitudes — so we only *score* direction and themes.
- **No fake precision** — percentages are hidden below minimum sample sizes (§6), confidence intervals are always shown.
- **Failures are published** alongside hits (US-6.2); outcome records are append-only (data-model invariant 7).

---

## 1. What counts as a hit

### 1.1 The scored unit

The scored unit is one **issued verdict** (`verdicts.outcome = 'verdict'`). A verdict makes exactly two scorable claims, per adapter-contract §3:

| Claim | Content | Source |
|---|---|---|
| **Direction** | Winning variant's majority stance: `support` vs `oppose` | Ensemble modal direction |
| **Top themes** | Top-3 objection/support themes | ≥2/3-overlap themes across runs |

`NO_CONSENSUS` verdicts are **abstentions**: never scored, never in any hit-rate denominator, but counted and disclosed (§6). A high abstention rate is a product-health signal, not an accuracy dodge.

### 1.2 Ground-truth coding

At the end of the prediction window (§3), the real-world outcome is coded from outcome evidence (post-window collector pull over the same topic, and/or client ground truth per §7) into the same schema:

- **True direction** `D_true ∈ {support, oppose, mixed}` — majority polarity of outcome-corpus reaction to the decision as executed. `mixed` = neither polarity exceeds 55% of coded reaction items.
- **True theme set** `T_true` — themes present in ≥5% of coded outcome items, capped at the top 10 by prevalence. Themes are coded with the same labeling taxonomy used for verdict objections.

Coding is done by a human analyst **blinded to the prediction** (§4.4), with a second coder on a ≥20% sample (§5.3).

### 1.3 Hit definitions

**Direction-level:**
```
direction_hit = 1  if D_pred == D_true
              = 0  if D_pred != D_true  (including D_true = mixed)
```
A `mixed` real world scores 0 against any committed direction — we claimed a direction, the crowd didn't have one. This is deliberate: it penalizes false confidence.

**Theme-level:** with predicted theme set `T_pred` (the verdict's top-3) and true set `T_true`, a predicted theme matches a true theme when both label and polarity agree (same underlying concern, same valence — adjudicated by the coder, not string equality):
```
theme_score = |T_pred ∩ T_true| / |T_pred ∪ T_true|        (Jaccard)
theme_hit   = 1  if theme_score ≥ 0.5
```

**Per-verdict outcome** (maps to `outcome_logs.outcome` enum):

| `outcome_logs` value | Condition |
|---|---|
| `hit` | direction_hit = 1 **and** theme_score ≥ 0.5 |
| `partial` | exactly one of {direction_hit = 1, theme_score ≥ 0.5} |
| `miss` | neither |

The structured scores (direction_hit, theme_score, theme sets, coder id) live in the Langfuse dataset item (§5); `outcome_logs` keeps the rollup enum + notes, and stays append-only.

### 1.4 Worked example — one verdict

Pricing decision, "raise plan to ₹499". Verdict: direction `oppose`; `T_pred = {price too high, subscription fatigue, competitor cheaper}`.

Window closes; outcome corpus coded blind: `D_true = oppose`; `T_true = {price too high, competitor cheaper, feature gating}`.

- direction_hit = 1
- |T_pred ∩ T_true| = 2, |T_pred ∪ T_true| = 4 → theme_score = 0.50 → theme_hit = 1
- **Outcome: `hit`.**

If instead `T_true = {price too high, feature gating, support queues}`: theme_score = 1/5 = 0.20, direction still correct → **`partial`**.

---

## 2. Theme-level convergence rubric (the ≥70% target)

PBR §16 sets the headline target: **≥70% theme-level convergence on logged outcomes**. Precise definition:

```
theme_convergence_rate = (# scored verdicts with theme_score ≥ 0.5)
                         ─────────────────────────────────────────────
                         (# scored verdicts in the measurement window)
```

Rules:

1. **Denominator** = all verdicts with a logged outcome of `hit`, `partial`, or `miss` in the window. Abstentions (`NO_CONSENSUS`) are excluded and reported separately. Verdicts still inside their prediction window are excluded (they are pending, not successes).
2. **Numerator** counts theme-level convergence only — direction does not count toward the headline number. A verdict can be direction-right and still fail the headline metric.
3. `partial` outcomes count in the denominator; they count in the numerator only if their theme_score ≥ 0.5 (i.e. they were theme-right/direction-wrong).
4. Always published with **n** and a **Wilson 95% confidence interval**. No interval, no number.
5. Target ≥ 0.70 measured on a **rolling 90-day** window, overall and per vertical (§3). A config change that drops the corpus theme rate by >2 pts is a release blocker (§5.4).

**Worked example — aggregate:** 40 verdicts scored in the window; theme_score ≥ 0.5 in 29.

- theme_convergence_rate = 29/40 = **0.725**
- Wilson 95% CI: center = (p̂ + z²/2n) / (1 + z²/n) with z = 1.96 → 0.705; half-width ≈ 0.134 → **CI (0.57, 0.84)**
- Published as: *"Theme-level convergence 72.5% (n=40, 95% CI 57–84%). Target ≥70%: met on point estimate, not yet statistically separated from it."* — that last clause is mandatory honesty, not optional phrasing.

Secondary metrics (published alongside, never as the headline): direction-only hit rate, full-`hit` rate (both dimensions), abstention rate, mean theme_score.

---

## 3. Prediction windows per vertical

The window is **declared at verdict issuance** and is immutable (outcome records are append-only; there is no mechanism to extend a window retroactively). Outcome coding happens at window close, with one optional late re-check logged as a separate record.

| Vertical (PBR §7) | Window | Rationale |
|---|---|---|
| Marketing & advertising | 14 days | Campaign discourse peaks and decays fast |
| Product & design (launch/concept) | 28 days | Reaction survives past launch-week noise |
| Pricing | 28 days | Backlash vs acceptance takes weeks to settle |
| PR & crisis | 96 hours | Crisis reaction is measured in days, not weeks |
| Policy & advocacy | 8 weeks | Aligns with F-34's 2–8 week forecasting horizon; minimum 2 weeks |
| Content & media | 21 days | Covers release + second-weekend/second-episode effect |
| Hiring & employer brand | 6 weeks | Employer-brand reaction is slow-moving |
| Investor & corporate comms | 10 days | Event-driven; window ends before the next scheduled disclosure |
| Local business | 30 days | Low-volume discourse needs time to accumulate ≥ the §4.1 outcome floor |
| Personal / self-serve (F-26) | 14 days | Same dynamics as marketing |

A verdict whose real-world decision is never executed (client shelved it) is logged as **unscorable** in notes — it enters no denominator. We score predictions about things that happened, not hypotheticals that stayed hypothetical.

---

## 4. Backtest protocol

A backtest replays a historical decision event through the full pipeline with the future cut off. Backtests seed the public accuracy page before organic outcome volume exists (F-18, roadmap M4).

### 4.1 Event selection

- Events come from a **pre-registered candidate list** per vertical (e.g. "all consumer product launches in category X in quarter Q meeting the data floor"), written down before any backtest in that batch runs.
- Inclusion floors: ≥500 collectable pre-event items across ≥2 sources, and a post-event outcome corpus ≥200 items (below that, theme coding is noise).
- Every candidate **dropped after registration is logged with its reason** on the accuracy page. Silent dropping is cherry-picking; logged dropping is methodology.
- No event may be backtested twice with different configs and only the better run published (config is frozen per §4.3 before the run).

### 4.2 Seeding (the time cut-off)

Let `T0` = the moment the real-world decision became public.

1. Collectors run with `time_window_days` bounded so that **no item with `published_at ≥ T0 − 24h`** enters the grounding set (24h buffer against embargo leaks and pre-announcement rumor).
2. The grounding snapshot is frozen: item set hashed (SHA-256 over sorted `item_id`s), stored in MinIO with the reality-seed document. This hash is the backtest's tamper-evidence.
3. Persona panel and variants are built from the frozen snapshot only — the standard handoff pipeline, no special casing.

### 4.3 Sealed prediction

The simulation runs at the standard cost-optimized config (PBR §11.1) unless the pre-registration specifies otherwise. The resulting verdict (direction, top-3 themes, agreement_score, config, snapshot hash) is **sealed**: serialized, hashed, timestamped, and stored before any outcome data is touched. The seal is published with the backtest.

### 4.4 Blinding

- The analyst **coding the outcome** (§1.2) does not see the sealed verdict until coding is complete and committed.
- The engineer **running the simulation** does not see post-`T0` data (the time cut-off makes this structural, not honor-system).
- Second-coder sample (§5.3) is coded blind as well.

### 4.5 Scoring

Identical to live scoring (§1, §2) — same rubric, same thresholds. Backtests and live outcomes are pooled in the corpus but **always disaggregable** on the public page; a platform that looks accurate only on replays must be visibly so.

---

## 5. Langfuse eval harness (F-18 infrastructure)

Langfuse holds the validation corpus as **datasets**; scoring results attach as **scores** on dataset runs. Concepts below use Langfuse's dataset/score/experiment model — exact SDK calls per the official Langfuse docs, not reproduced here.

### 5.1 Datasets

| Dataset | One item = | Visibility |
|---|---|---|
| `validation-corpus` | A sealed prediction + its coded outcome: input = {snapshot hash, variant label, sealed verdict}, expected_output = {D_true, T_true, coder ids} | Global, append-only |
| `backtest-registry` | Pre-registered backtest candidates incl. dropped-with-reason | Global |
| `calibration-surveys` | Real poll/survey benchmarks for F-32 distribution checks | Global |
| `ground-truth-{tenant}` | F-23 client uploads, coded | **Per-tenant only** — never pooled (US-6.3, §7) |

### 5.2 Scorers

| Scorer | Type | Computation |
|---|---|---|
| `direction-match` | Deterministic | §1.3 formula |
| `theme-jaccard` | Deterministic | §1.3 Jaccard over coded theme sets |
| `coder-agreement` | Deterministic | Cohen's κ between primary and second coder on the ≥20% double-coded sample |
| `claim-evidence-consistency` | LLM-as-judge (`report-model`) | Per PBR §8.3: each report claim checked against its citations before publish |
| `calibration-divergence` | Deterministic | Total-variation distance between agent stance distribution and the matched survey benchmark (F-32) |

### 5.3 Thresholds

| Gate | Threshold | On failure |
|---|---|---|
| Outcome accepted into corpus | coder κ ≥ 0.70 (sampled) | Entire batch re-coded; persistent low κ → taxonomy revision |
| Verdict issuance (F-32) | calibration TV distance ≤ 0.15 | Verdict blocked, panel flagged for regeneration (US-6.4) |
| Headline accuracy (PBR §16) | theme_convergence_rate ≥ 0.70, rolling 90d | Release/marketing claims frozen until recovered |
| Direction secondary metric | ≥ 0.75 target | Reported, not gating |

### 5.4 Experiment gate for engine changes

Any change to the simulation core (model alias, persona construction, ensemble config, prompt set) must run the full `validation-corpus` as a Langfuse experiment before rollout: **ship only if corpus theme_convergence_rate does not drop by more than 2 points** versus the current production config on the same items. This is the mechanism behind PBR §11.1's "any further cuts require validation-corpus evidence."

---

## 6. Publication rules (public accuracy page, US-6.2)

| Rule | Value | Why |
|---|---|---|
| Percentage display floor | **n < 10 → no %, counts only** | Fixed by phase-7 P7-T7 / mt-07 step 8: no fake precision |
| Headline rate floor | **n ≥ 30** scored verdicts overall | Below that the Wilson interval is wider than the claim |
| Per-vertical cell floor | **n ≥ 20**, else "insufficient data" + counts | Vertical claims are sales claims; hold them to a floor |
| Interval | Wilson 95% CI next to every % | §2 rule 4 |
| Failures | Every `miss` and `partial` listed with its one-line post-mortem, next to the hits | US-6.2 acceptance criterion |
| Abstentions | `NO_CONSENSUS` count + rate disclosed | They are excluded from denominators, never from disclosure |
| Backtests | Snapshot hash + sealed-verdict hash + pre-registration note published per backtest | Tamper-evidence (§4.2–4.3) |
| Dropped candidates | Listed with reason | Anti-cherry-picking (§4.1) |
| Corrections | Appended, never edited — `outcome_logs` is append-only (data-model invariant 7) | The page's credibility is the schema's |
| Coverage gaps | Each published backtest links its grounding coverage map (disclosed gaps per US-2.1) | Never oversell (PBR §15) |
| Magnitudes | The page never shows predicted CTRs, vote shares, or revenue — we don't predict them, so we can't be scored on them | PBR §2 principle 4 |

---

## 7. Client ground truth (F-23) → scoring

US-6.3: surveys via Formbricks, recordings transcribed via Whisper, **used only for that tenant's calibration**.

### 7.1 Ingestion floors

| Source | Minimum to be scorable |
|---|---|
| Formbricks survey | ≥30 completed responses per decision question |
| Whisper transcripts | ≥5 interviews per decision question |

Below the floor: stored, shown to the tenant, excluded from scoring (same no-fake-precision rule as §6).

### 7.2 Coding and precedence

Uploads are coded into the same `{D_true, T_true}` schema as public outcome corpora (§1.2), by the tenant's analyst or ours, and land in `ground-truth-{tenant}` (§5.1). When both a public outcome corpus and client ground truth exist for the same verdict, **client ground truth takes precedence** for that tenant's scoring — it measures the client's actual customers, which is the reaction the client paid to predict. Both codings are kept; the public-corpus coding still feeds the global corpus.

### 7.3 What tenant ground truth feeds

1. **Tenant's prediction-vs-reality log** (US-6.1): verdicts scored per §1, visible in that workspace's Validation Center.
2. **Tenant calibration** (F-38): persona-proportion tuning proposals from logged outcomes, human-approved, full drift audit trail.
3. **Never** the public accuracy page or the pooled `validation-corpus` — unless the tenant explicitly opts into anonymized contribution, revocable (same withdrawal semantics as F-45 benchmarks).

### 7.4 Compliance

Client ground truth is the client's own customer data (surveys, interviews) — consent and lawful basis are the client's responsibility; CrowdLens persists no platform-user PII from these uploads (PBR §11 compliance row), stores transcripts and coded themes per-workspace isolated, and never trains shared models on them.

---

*End of accuracy methodology v0.1. The headline number on the public page is only as good as this document's enforcement — when in doubt, publish the failure.*
