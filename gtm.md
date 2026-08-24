# CrowdLens — Go-To-Market Plan

**Version:** 0.1 (Draft) · **Date:** 2026-08-25 · **Companion to:** readme.md (PBR) v0.1, features.md v0.1, user-stories.md v0.1

The commercial layer of the PBR: who buys, through which channel, on what terms, and how we know it's working. Everything here inherits the Product Principles (PBR §2) — GTM claims follow the same grounded-or-nothing rule as product output. We never sell precision we don't have; the accuracy page (F-18) is our loudest asset and it only works if it's honest.

**Constraints this plan operates under** (from PBR): verdict cost ≤ $25 all-in (§11), time-to-verdict < 24h (§16), adaptive ensemble 3–7 runs at ≥80% agreement (§11.1), no election-result prediction or trading signals (§1), DPDP + GDPR, no platform-user PII persisted (§9).

---

## 1. ICP Definition & Buying Triggers

ICP per persona (PBR §3). "Trigger" = the observable event that makes a buyer search for us this week, not a generic pain.

| Persona | ICP (narrow) | Buying trigger | What they buy first | Tier |
|---|---|---|---|---|
| **Solo Consultant / Creator** | Course creators, cohort-based educators, indie consultants with an audience; India + global English | Launch in 2–6 weeks; last launch underperformed; price-point or topic anxiety | One guided project: "will this offer land, at this price?" | Starter |
| **Agency Strategist** | 10–100 person PR / digital / performance agencies; 3+ active retainers; already marking up research or social listening | Pitch lost to a data-heavy competitor; client asks "how do you know?"; crisis client needs same-day read | Multi-client workspace + one white-label client report (F-19) | Agency |
| **Enterprise Comms / Insights Lead** | 500+ employee consumer brands, banks, conglomerates (India-native depth is the wedge); existing insights budget ≥ $100K/yr | Pre-publication statement/apology testing; board asks for evidence on a repositioning; compliance blocks a vendor | Pre-publication testing under SSO + audit log (AUTH-5/7); accuracy review | Enterprise |
| **Product Manager / Designer** | Consumer-app product teams shipping ≥ monthly; running surveys today and hating the lag | Naming/packaging/feature decision deadlocked internally; survey came back inconclusive | Concept reaction test with aspect-level feedback (F-24 when live) | Agency/Enterprise |
| **Campaign / Policy Analyst** | Advocacy orgs, policy comms teams — **within legal guardrails only** (§9, §15); §126A lock enforced | Message-framing decision outside restricted windows; issue-mining need ahead of a campaign | Issue mining + message testing with the guardrails visible, not hidden | Enterprise |

**Disqualifiers (we walk away):** election-result prediction, trading signals, astroturfing, anything requiring PII of platform users. Saying no is positioning (PBR §1, §15).

---

## 2. Channel Strategy per Tier

| Tier | Motion | Channels | CAC logic |
|---|---|---|---|
| **Starter** (self-serve) | Product-led. No sales touch. | F-26 viral mini-tool (§4), vertical template SEO per F-22 pack ("test your course idea before launch"), creator communities, the public accuracy page as proof content | Near-zero CAC target; the mini-tool and template pages do the work. Paid spend only retargets mini-tool users |
| **Agency** (outbound) | Founder-led outbound → pilots (§3) → self-serve expansion | Warm intros, LinkedIn founder content showing real verdict reports (with receipts), agency communities/events, 1:1 demos on the Simulation Theater — the wow moment (PBR §6.1) sells itself | Founder time is the CAC; target payback < 3 months on Agency MRR |
| **Enterprise** (sales) | Consultative, security-first (§6) | Direct outreach to comms/insights leads, agency referrals (agencies serve enterprises), India-native data depth as the wedge pitch, accuracy page as the leave-behind | Long cycle (2–6 mo) tolerated — ACV $60–180K/yr justifies it |

**Channel rule:** every channel leads with receipts. A demo that doesn't show citations and convergence scores is off-brand.

---

## 3. M4 Pilot Program — "5 Agency Pilots" Made Concrete

Roadmap M4 (wks 11–14) commits to 5 agency pilots. This is the operating plan.

### 3.1 Who

Target profile: 10–100 person PR/digital agencies, 3+ active retainers, at least one consumer-brand client, Mumbai/Bangalore/Delhi-NCR weighted (India pack is M4 scope), plus up to 2 English-market agencies for contrast. Build a list of **50** agencies, expect ~25 conversations, ~10 demos, **5 signed pilots**.

### 3.2 Outreach motion (founder-led)

1. **Week 1 of M4:** list of 50 built; warm-intro requests out; founder LinkedIn series starts — one real anonymized verdict report per week, receipts visible.
2. **Pitch:** "Bring one live client decision. We'll test it against a simulated market built from real cited data, same-day verdict, white-labeled with your logo."
3. **Demo:** Simulation Theater live on their seed document (never a canned demo) → verdict page → cost ledger block (§8.1 — the ledger is the agency markup transparency story).
4. **Close:** pilot agreement (below) signed in the same week as the demo or it's dead — no zombie pilots.

### 3.3 Pilot terms

| Term | Value |
|---|---|
| Duration | 6 weeks |
| Price | Agency tier at 50% (₹12,500 / ~$150 mo equivalent) — paid, not free; free pilots don't convert and don't give real feedback |
| Credits | 60 credits/mo as per Agency tier; overage at standard pack rates (§5) |
| Commitment from agency | ≥3 real client projects run through the platform; 30-min feedback call weekly; permission to publish one anonymized case study |
| Commitment from us | Named Slack channel, same-day bug triage, white-label exports (F-19) enabled from day one |
| Exit | Either side can end at week 3 with no further obligation |

### 3.4 Success criteria (measured at week 6)

- **Conversion:** ≥3 of 5 pilots convert to full-price Agency within 30 days of pilot end.
- **Activation:** each pilot agency averages ≥1 weekly active project per seat (feeds PBR §16 metric).
- **Evidence:** ≥3 publishable case studies (anonymized ok); ≥2 logged prediction-vs-reality outcomes feeding the validation corpus (F-18).
- **Referral:** ≥2 of 5 pilots introduce us to another agency or an enterprise client.

A pilot that runs projects but won't convert or refer is a failed pilot — log why, don't relabel it a win.

---

## 4. F-26 Viral Mini-Tool Funnel Mechanics

The dating-profile/consumer mini-tool (F-26) is the top of the self-serve funnel. It is a **narrowed, rate-limited slice of the real pipeline**, not a toy — the honesty principles apply in full.

### 4.1 Flow

1. **Input:** user pastes profile text / uploads screenshots; picks the decision ("which photo order?", "this bio vs that bio"). No signup required.
2. **Grounding-lite:** reduced collector run (public dating-adjacent discussion data only) → small persona panel. Clearly labeled: *sampled public opinion, not a survey of your matches.*
3. **Mini-simulation:** 2 variants, reduced ensemble (never below 3 runs — §11.1 floor holds even here).
4. **Verdict card:** one shareable image/page — ranked variants, top objections with quotes, confidence meter, watermark. **No invented scores**: direction and themes only, exactly like the main product (PBR §1).
5. **Share loop:** card carries "Tested against a simulated crowd — CrowdLens" + link. Per-link analytics (§8.2 share-link infra reused).
6. **Upsell:** "This was 2 variants on public data. Test any decision — pricing, launch, positioning — with full grounding." → Starter signup, first project discounted or bonus credits.

### 4.2 Funnel mechanics

| Stage | Mechanic | Metric |
|---|---|---|
| Reach | Shared verdict cards on social; each card is an ad with receipts | Cards shared / verdicts run |
| Conversion to tool | Link → landing → run in < 2 min, no signup | Completion rate of mini-run |
| Activation | Email capture to *save* the verdict (optional, magic link — AUTH-1) | Save rate |
| Monetization | Starter signup with bonus credits | Mini-tool → Starter conversion % |
| Loop | K-factor target > 0.3 at launch; iterate share card creative, not the science | K-factor |

### 4.3 Guardrails (non-negotiable)

- Every output labeled as simulated/sampled; the F-18 accuracy page links from the mini-tool footer.
- No PII persisted from uploads (DPDP/GDPR — PBR §9); uploads are processed and discarded on a stated schedule.
- Rate limits + spend caps via the same LiteLLM virtual-key machinery (AUTH-4) — the free funnel cannot blow the LLM budget.
- The mini-tool is one F-22 vertical template ("dating"), not a fork of the codebase.

---

## 5. Pricing Page & Credit-Pack Logic

### 5.1 Page structure (mirrors PBR §14 exactly)

| Tier | Price | Credits | Page emphasis |
|---|---|---|---|
| Starter | ₹4,999 / $59 mo | 10/mo | "Test one decision a week." Guided wizard, community persona panels |
| Agency | ₹24,999 / $299 mo | 60/mo | White-label exports, 5 seats, ad testing (F-13) — the default choice, visually highlighted |
| Enterprise | Custom ($5–15K/mo) | Negotiated | SSO, on-prem option (F-21), private panels, SLA, accuracy reports |

**1 credit = 1 full decision cycle** (grounding + branched ensemble simulation + verdict), per §14. The pricing page says this in plain words and links the accuracy page. No per-feature nickel-and-diming above the fold.

### 5.2 Credit-pack (overage) logic

- **Rollover:** unused credits roll 1 month, then expire. Roll-forever destroys the metered model; no-rollover feels punitive.
- **Add-on packs:** sold in 10-credit packs at a **premium to in-plan per-credit rate** (≈20% above plan rate) — overage should nudge upgrades, not replace them. Lago meters all of it (PBR §4.2).
- **Hard caps stay hard:** F-08 spend caps are a product safety feature, not a billing edge case. A project that hits its cap halts; topping up is an explicit user action.
- **Never below cost floor:** no discount, pilot, or promo prices a credit below grounded all-in cost. The $25/verdict figure in §11/§16 is the **worst-case cap**, not the typical cost (§11.1 puts the simulation layer at ~$2–3); pack pricing must be set against *measured* typical cycle cost from Langfuse/Lago ledgers, and revisited monthly.
- **Enterprise:** credits become an internal metering concept; contract is ACV + committed volume + overage schedule.

---

## 6. Enterprise Sales Motion

Cycle assumption: 2–6 months, security-led. The motion is built so our honesty artifacts do the selling.

1. **Land (weeks 0–4):** intro via agency referral or direct outreach → discovery around a live pre-publication or repositioning decision → paid proof-of-concept (one decision cycle, real data, exec-readable verdict, F-09).
2. **The asset (weeks 2–8, in parallel):** the **public accuracy page (F-18)** is the leave-behind — validation corpus, prediction-vs-reality log, backtest library, methodology. It works because it shows misses too. No enterprise deck outranks it.
3. **Security & compliance review (weeks 4–10):** standard pack prepared once, reused: encryption at rest, per-workspace isolation, audit logs (AUTH-7), SSO SAML/OIDC (AUTH-5), DPDP + GDPR posture, no platform-user PII persisted, citation-based transparency, AGPL fork isolation and licensing position (PBR §11 Licensing, §15). India enterprises get the §9.2 coverage map with gaps *disclosed* — never oversell (§15).
4. **On-prem track (weeks 6+, when triggered):** F-21 profile — single docker-compose/Helm, local models via vLLM/Ollama, GPU sizing tiers 24GB/48GB/2×80GB, license-key activation (AUTH-8). On-prem is offered when data-residency or air-gap is a hard requirement, not as a discount lever. PBR §16 target: ≥3 on-prem deployments closed in the first 2 quarters.
5. **Close & expand:** start with comms/insights team, committed credit volume; expand via monitoring subscriptions (F-16/17), BI connectors (F-43), and additional business units. Expansion revenue is a §16 metric — the land-and-expand path is the model, not an accident.

---

## 7. 90-Day Launch Plan

Day 0 = start of M4 (PBR §13, week 11). Weeks below are launch weeks; product-scope items reference roadmap milestones, not new commitments.

| Week | Milestones |
|---|---|
| W1 | Pilot target list (50 agencies) built; founder LinkedIn series live; pricing page published (Starter + Agency purchasable, Enterprise "talk to us"); F-22 template pages drafted for 3 verticals (course launch, PR crisis, product design) |
| W2 | First 10 pilot conversations; demo script locked (Theater → verdict → cost ledger); analytics wired end-to-end (signup → first verdict → credit burn) |
| W3 | First pilot signed; mini-tool (F-26) landing + waitlist page live; India data pack messaging added to site (multilingual, tier-2/3 depth) |
| W4 | 2 pilots signed; first pilot's first client project runs; weekly pilot feedback calls begin; accuracy page skeleton published with methodology (honest even while empty) |
| W5 | Mini-tool public launch; share-card loop instrumented (K-factor); 3rd pilot in negotiation; first backtest results published to accuracy page (M4 scope) |
| W6 | 3–4 pilots signed; mini-tool iteration #1 on share creative; Starter self-serve funnel live with credit packs purchasable |
| W7 | Pilot mid-point reviews (week-3 exit checkpoint honored); agency case study #1 drafted; enterprise outreach list built from pilot referrals |
| W8 | M4 closes: 5 pilots signed target; case study #1 published; first enterprise discovery calls held |
| W9 | M5 assets begin: monitoring/alerts (F-16/17) pitched to converting pilots as expansion; security review pack finalized |
| W10 | ≥1 pilot converted to full-price Agency; accuracy page populated with live prediction-vs-reality log; on-prem conversation opened with ≥1 enterprise prospect |
| W11 | Enterprise POC #1 (paid, one decision cycle); public API (F-20) announced to agency tier |
| W12 | Pilot cohort retrospective: conversion, activation, referral scorecard vs §3.4; mini-tool funnel review; pricing/credit-pack rates revisited against measured cycle cost |
| W13 | 90-day review against §8 metrics below; go/no-go on paid acquisition for Starter; enterprise pipeline report (target: ≥3 active opportunities, ≥1 in security review) |

---

## 8. Metrics per Stage (tied to PBR §16)

Every GTM stage reports a §16 metric alongside its funnel metric — the product metrics *are* the go-to-market proof.

| Stage | Funnel metric | §16 product metric gate |
|---|---|---|
| **Awareness** | Mini-tool runs, share K-factor, template-page organic traffic | Accuracy page live and honest — target ≥70% theme-level convergence on logged outcomes (§16); we don't scale spend on a page that can't show this |
| **Signup / first project** | Mini-tool → Starter conversion; time-to-first-verdict per user | Time-to-verdict < 24h (§16) — a first verdict that takes longer kills self-serve conversion |
| **Activation** | Projects per workspace; guided-wizard completion | Cost-per-verdict < $25 (§16) — verified per project from the Lago/LiteLLM ledgers, not estimated |
| **Paid conversion** | Pilot → Agency conversion (target ≥3/5); trial → paid % | Verdict accuracy trend on the logged-outcomes corpus — quoted in every sales conversation |
| **Expansion** | Weekly active projects per agency seat; credit burn rate; expansion revenue (all §16 verbatim) | Credit burn growth without accuracy drift — more runs must hold the ≥70% convergence bar |
| **Enterprise** | POCs run, security reviews cleared, ACV pipeline | On-prem deployments closed in first 2 quarters ≥ 3 (§16) |

**Review cadence:** funnel metrics weekly (§7 cadence); §16 gates monthly; any gate missed two months running triggers a plan revision, not a metric redefinition.

---

## 9. Open Tensions (tracked, not hidden)

- **Starter unit economics:** Starter prices a credit at ≈ $5.90 while §11/§16 allow up to $25 all-in cost per verdict. The model only works if *measured typical* cycle cost stays far below the cap (§11.1 suggests the simulation layer is ~$2–3, but grounding volume is the variable). Pack pricing and tier limits must be set from real ledger data within the first 90 days — see §5.2 and W12.
- **Free-funnel cost exposure:** F-26 runs consume real collector + LLM budget. Rate limits and virtual-key caps (AUTH-4) are the control; if K-factor < 0.3 and cost-per-acquired-Starter exceeds 3 months of Starter MRR, the mini-tool gets narrowed, not killed — it's also the dating vertical template.
- **Accuracy page timing:** the page is the enterprise asset (§6) but only becomes persuasive with logged outcomes. Publishing the methodology skeleton in W4 with empty-but-honest data is deliberate; scaling enterprise outreach before ≥2 logged outcomes exist (§3.4) is premature.

---

*End of GTM plan v0.1 — companion artifacts: readme.md (PBR), features.md, user-stories.md, ai-dev/ build kit.*
