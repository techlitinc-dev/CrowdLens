# Phase 6 — India Data Pack & Multilingual

**Goal:** Hindi/Marathi/regional grounding + multilingual analysis and reports — the differentiation layer (PBR §9.2). **Gate:** `manual-tests/mt-06-india.md` green.

## Prerequisites
- Phase 5 green. Human obtains: Telegram API credentials (`TELEGRAM_API_ID`, `TELEGRAM_API_HASH` from my.telegram.org). Google Play reviews need no key (public data).

## Tasks

### P6-T1 — Telegram collector (public channels only)
Via the BettaFish fork if supported, else a new collector in the adapter service using Telethon: given channel handles or keyword-discovered public channels, pull messages + public comments. Contract shape identical to other sources (`source: "telegram"`). PII strip: sender names/ids dropped at the boundary. **Public channels only — never join groups, never scrape private data.**
**Done when:** mt-06 step 2 — real rows from 3 Indian news/deals channels in `collected_items`.

### P6-T2 — Google Play / App Store review collector
`source: "play_store"` / `"app_store"` via a scraper library (google-play-scraper or equivalent — pin version). Fields map to the standard item shape; review id → item_id hash. Per-app daily cap 2,000 reviews (politeness + quota).
**Done when:** unit tests + mt-06 step 3.

### P6-T3 — Language detection + pipeline
At ingestion (adapter): fasttext-based langid (`langid` field already exists — now populate reliably for `hi`, `mr`, `hi-Latn` Hinglish). Rule: Hinglish (romanized Hindi) detected → `language: "hi-Latn"`, not `en`.
**Done when:** fixture test set (provided in task file's tests) ≥90% correct on 30 labeled samples.

### P6-T4 — Multilingual analysis & personas
Analysis prompts (`report-model`) updated: instructions to analyze hi/mr/hi-Latn text natively (no translate-first), quotes kept in original script with EN gloss. Persona panels gain a `languages[]` per archetype; language samples must match the archetype's languages.
**Done when:** mt-06 step 5 — Marathi quotes appear verbatim (not translated) with glosses in the baseline report.

### P6-T5 — Bilingual reports
Report rendering gains `language: "en" | "hi" | "mr"` param: UI chrome + generated narrative translated via `report-model`; **quotes and citations always stay original-script**. Devanagari font (Noto Sans Devanagari) in tokens; PDF export embeds it.
**Done when:** mt-06 step 6 — same report in EN and HI, citations identical.

### P6-T6 — Coverage map UI
Grounding Console gains region/language filters and a coverage disclosure panel: per-source item counts by region/language, and an explicit "no coverage" list (e.g. ShareChat) — this disclosure ships in every report as a standard block (`markdown` block, auto-generated).
**Done when:** mt-06 step 7.

## Compliance checklist (human verifies in mt-06)
- DPDP: no PII stored (re-run the paranoia PII check on telegram/play_store rows)
- Every report carries the coverage disclosure
- Telegram: public channels only

## Stop conditions
- Telethon auth requires a phone number you don't have → stop, hand to human (it's a one-time manual session setup).
- langid accuracy <90% on the fixture set → stop, report per-language confusion matrix.

## Manual test
`../manual-tests/mt-06-india.md`
