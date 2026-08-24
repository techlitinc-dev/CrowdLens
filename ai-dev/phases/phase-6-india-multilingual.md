# Phase 6 — India Data Pack & Multilingual

**Goal:** Hindi/Marathi/regional grounding + multilingual analysis and reports — the differentiation layer (PBR §9.2). **Gate:** `manual-tests/mt-06-india.md` green.

## Prerequisites
- Phase 5 green. Human obtains: Telegram API credentials (`TELEGRAM_API_ID`, `TELEGRAM_API_HASH` from my.telegram.org). Google Play reviews need no key (public data).

## Tasks

### P6-T1 — Telegram collector (public channels only)
Via the BettaFish fork if supported, else a new collector in the adapter service using Telethon: given channel handles or keyword-discovered public channels, pull messages + public comments. Contract shape identical to other sources (`source: "telegram"`). PII strip: sender names/ids dropped at the boundary. **Public channels only — never join groups, never scrape private data.**
**Done when:** mt-06 step 2 — real rows from 3 Indian news/deals channels in `collected_items`.

**Files:**
- First, inspect `forks/bettafish/` for existing Telegram support (read-only). If present: config-only enablement, no fork patches (R6).
- Else, new files in the adapter service:
  - `services/adapters/collectors/telegram.py` — the collector.
  - `services/adapters/collectors/pii.py` — shared boundary stripper (extend, don't duplicate, if a per-source stripper already exists from phase 1).
  - `tests/adapters/test_telegram_collector.py`.

**Key functions:**
```python
class TelegramCollector:
    async def collect(self, channels: list[str], keywords: list[str],
                      time_window_days: int) -> AsyncIterator[RawMessage]
def to_contract_item(msg: RawMessage, channel: str) -> dict   # adapter §1 item shape
def strip_pii(msg: RawMessage) -> RawMessage                   # drops sender name/id/username
```
Telethon usage: `TelegramClient` with an API-id/hash session (session file from the human's one-time login — commit the `.gitignore` entry for it, never the file). Iterate channel history + discussion-thread replies via the official Telethon docs (`iter_messages`, message replies); do not hand-roll MTProto calls.

**Contract mapping:** `item_id = sha256("telegram" + channel + message_id)`; `url = https://t.me/{channel}/{message_id}` (must open without login — mt-06 step 3 spot-checks this); `metrics` carries views/forwards only, never author fields.

**Tests** (`pytest tests/adapters -k telegram`):
- `test_item_shape_matches_contract` — validates against the adapter §1 item schema.
- `test_pii_stripped_at_boundary` — raw fixture with sender fields → output has none.
- `test_group_and_private_links_rejected` — `t.me/+...` invite links and group ids raise `INVALID_INPUT` before any network call.
- `test_floodwait_backoff` — mocked `FloodWaitError` → waits, resumes, no retry storm.

**Edge cases:** channel renamed/deleted mid-run → source marked failed honestly (frontend rule 4), other sources continue; forwarded duplicates (same text, different channels → different `item_id`s — acceptable, dedupe happens in analysis); media-only messages (no `text`) → skipped and counted in `progress`, never inserted with empty text; channels with comments disabled → messages only, no failure.

### P6-T2 — Google Play / App Store review collector
`source: "play_store"` / `"app_store"` via a scraper library (google-play-scraper or equivalent — pin version). Fields map to the standard item shape; review id → item_id hash. Per-app daily cap 2,000 reviews (politeness + quota).
**Done when:** unit tests + mt-06 step 3.

**Files:**
- `services/adapters/collectors/play_store.py` — uses the Python `google-play-scraper` package (pin in `services/adapters/requirements.txt`; API per its official docs — fetch reviews paged by continuation token, newest first).
- `services/adapters/collectors/app_store.py` — Apple has a public RSS customer-reviews endpoint; use it before any third-party scraper.
- `tests/adapters/test_play_store_collector.py`, `test_app_store_collector.py`.

**Key functions:**
```python
def collect_reviews(app_id: str, country: str = "in",
                    lang: str | None = None, max_items: int = 2000) -> Iterator[dict]
def to_contract_item(review: dict, app_id: str) -> dict
```
`item_id = sha256(source + app_id + review_id)`; `url` = store listing URL (review permalink where available); `metrics = {score, thumbs_up}`; reviewer name/avatar dropped in `to_contract_item` (PII, mt-06 step 4). Daily cap enforced with a per-(app, day) counter persisted in the adapter's own store — reset at UTC midnight, never silently truncate mid-page (finish the current page, then stop).

**Tests:** `test_item_shape_matches_contract`, `test_item_id_stable_across_runs`, `test_daily_cap_enforced`, `test_reviewer_pii_dropped`, `test_continuation_pagination` (mocked pages).

**Edge cases:** app id typo → store 404 → `NOT_FOUND` envelope, not empty success; reviews with no text (rating-only) → skipped; duplicate reviews across pages (store pagination overlaps) → dedupe on `review_id` before hashing; `country="in"` but review language `hi` — keep, langid (P6-T3) handles it.

### P6-T3 — Language detection + pipeline
At ingestion (adapter): fasttext-based langid (`langid` field already exists — now populate reliably for `hi`, `mr`, `hi-Latn` Hinglish). Rule: Hinglish (romanized Hindi) detected → `language: "hi-Latn"`, not `en`.
**Done when:** fixture test set (provided in task file's tests) ≥90% correct on 30 labeled samples.

**Files:**
- `services/adapters/langdetect.py` — the only place language is decided. All collectors call it; none roll their own.
- `services/adapters/models/` — fasttext `lid.176.bin` (download in Dockerfile, not committed; 126 MB, suppress with a `.gitignore` entry).
- `tests/adapters/test_langdetect.py` + `tests/adapters/fixtures/langid_30.jsonl` — the 30 labeled samples.

**Key functions:**
```python
def detect_language(text: str) -> str   # "en" | "hi" | "mr" | "hi-Latn" | "und"
def _script(text: str) -> str           # "devanagari" | "latin" | "other"
```
Logic: fasttext predict (per fasttext docs — `model.predict(text.replace("\n", " "))`) → label + confidence. Post-rules: confidence < 0.6 or text < 20 chars → `"und"` (honest, downstream treats `und` as `en` for analysis but keeps the flag); label `hi`/`mr` + Latin script → `"hi-Latn"`; label `en` + Latin script stays `en` — the Hinglish rule applies only when fasttext says Indic under Latin script. **Do not use an LLM for langid** — cost, latency, and nondeterminism for a solved problem.

**Tests:** `test_fixture_accuracy` — ≥27/30 correct (the ≥90% gate; on failure print the per-language confusion matrix, which the stop condition requires). `test_short_text_returns_und`. `test_devanagari_never_latn`.

**Edge cases:** emoji-only / URL-only text → `und`; code-mixed Devanagari+Latin in one post → script of the majority of letters; fasttext thread-safety — load the model once at worker startup, predict is GIL-safe but don't lazy-load per request.

### P6-T4 — Multilingual analysis & personas
Analysis prompts (`report-model`) updated: instructions to analyze hi/mr/hi-Latn text natively (no translate-first), quotes kept in original script with EN gloss. Persona panels gain a `languages[]` per archetype; language samples must match the archetype's languages.
**Done when:** mt-06 step 5 — Marathi quotes appear verbatim (not translated) with glosses in the baseline report.

**Files:**
- Prompt templates live wherever phase 2 put them (BettaFish analyze prompt for the baseline; `app/services/handoff.py` for the persona panel) — edit in place, add the multilingual block below, don't fork the prompts per language.
- `app/services/multilingual.py` — shared helpers: gloss validation, language-tag checks.

**Prompt addition (intent, both prompts):** "Items may be in English, Hindi, Marathi, or Hinglish. Analyze in the original language — never translate before analyzing. Every verbatim quote stays in its original script, followed by a one-line English gloss in parentheses. Citation item_ids are untouched."

**Persona panel:** each archetype dict gains `languages: ["en","hi",...]` derived from the `language` field of the items that informed it. Note: this is an additive field on the handoff `persona_panel.archetypes[]` shape (adapter §2) — additive, never removing existing keys. Post-check: every `language_samples[i]` quote's source item has a `language` in the archetype's `languages[]`; mismatch → sample swapped for one that matches (never relabel the item).

**Post-checks (cheap, deterministic — the real gate):** each quote string in the analysis output must appear verbatim (substring) in its cited `collected_items.text`; each quote must be followed by a gloss; `hi-Latn` items must never surface as `language: "en"` downstream.

**Tests:** `test_quotes_verbatim_with_gloss`, `test_hinglish_not_english`, `test_persona_languages_match_samples`, `test_gloss_missing_reblocks_report`.

**Cheap-LLM failure modes (this is where a weak `report-model` hurts):**
- Silently translates quotes into English → caught by the verbatim-substring post-check → re-run once with a sterner prompt, then fail the report (never publish translated quotes).
- Invents fluent-sounding glosses that distort meaning → glosses are convenience only; the verbatim quote + item_id is the citation of record. Human eyeball in mt-06 step 5.
- Drops item_id references on non-Latin text (tokenizer artifacts) → citation check (adapter §1 hard rule) already rejects.
- Labels Hinglish items `en` in its narrative → post-check on the `language` field, not the model.

### P6-T5 — Bilingual reports
Report rendering gains `language: "en" | "hi" | "mr"` param: UI chrome + generated narrative translated via `report-model`; **quotes and citations always stay original-script**. Devanagari font (Noto Sans Devanagari) in tokens; PDF export embeds it.
**Done when:** mt-06 step 6 — same report in EN and HI, citations identical.

**Files:**
- `app/services/report_translate.py` — `translate_blocks(blocks: list[dict], target: str) -> list[dict]`. Walks the report-blocks tree; translates only narrative/markdown text fields; skips quote bodies, citation refs, block ids, and the auto-generated coverage block's counts.
- `frontend/src/theme/tokens.css` — add Noto Sans Devanagari to the font stack (headings + body); bundle the font locally (no Google Fonts request at runtime — offline/on-prem friendly).
- Export worker (phase 5): WeasyPrint `@font-face` pointing at the bundled font file so the PDF embeds it (mt-06 step 8 — no tofu boxes).
- `tests/core/test_report_translate.py`.

**Deterministic guards around the LLM call:** hash every quote body before translation, assert identical set after; extract all numerals before/after and compare multiset (no invented or "translated" numbers — PBR no-invented-numbers principle); citation refs compared by exact equality. Any mismatch → discard the translation, log, return 500 — never publish a mutated citation.

**Tests:** `test_translate_preserves_quotes_and_citations`, `test_translate_preserves_numerals`, `test_same_report_two_languages_same_block_ids`, `test_pdf_embeds_devanagari` (open the PDF with pypdf and assert the font is embedded — if pypdf's font API is uncertain, assert via the official pypdf docs rather than guessing).

**Edge cases:** `reports` DDL already constrains `language IN ('en','hi','mr')` with `UNIQUE(project_id, version, language)` — a HI render is a new row, the EN original is immutable; mixed-script narrative (English brand names inside Hindi text) is fine, don't force-transliterate; missing gloss on a legacy quote → translate anyway, gloss stays absent (don't invent one).

**Cheap-LLM failure modes:** model restructures markdown (dropped headings/tables) → compare block count and types before/after; model "improves" claims while translating → numeral/citation guards; Devanagari output sprinkled with Latin transliteration → acceptable, do not post-correct.

### P6-T6 — Coverage map UI
Grounding Console gains region/language filters and a coverage disclosure panel: per-source item counts by region/language, and an explicit "no coverage" list (e.g. ShareChat) — this disclosure ships in every report as a standard block (`markdown` block, auto-generated).
**Done when:** mt-06 step 7.

**Files:**
- Backend: the panel needs a per-source × language × region aggregate. That is a new read endpoint (e.g. `GET /grounding/{job_id}/coverage`) — **add it to `contracts/api-surface.md`'s extended-surface index first (R1), then implement.** Do not compute it client-side from paged items.
- `app/services/coverage.py` — `build_coverage(job_id) -> CoverageSummary` (pure SQL aggregate) and `render_disclosure_block(summary) -> dict` (the auto `markdown` block injected into every report at render time, before publish validation).
- `frontend/src/routes/grounding/CoveragePanel.tsx` + `frontend/src/components/CoverageDisclosure.tsx` — filters reuse the existing console's query params; disclosure panel is a fixed section, not dismissible.
- `tests/core/test_coverage.py`, `tests/frontend/CoverageDisclosure.test.tsx`.

**Disclosure content (fixed template, no LLM):** per-source counts by language/region; the "no coverage" list (ShareChat, WhatsApp per mt-06 step 7) rendered verbatim; one sentence: "Sources not listed had no items in this collection window." Template-generated so it can never hallucinate a count.

**Tests:** `test_coverage_counts_match_items_table`, `test_disclosure_block_in_every_rendered_report`, `test_no_coverage_list_present`, `test_empty_source_shows_zero_not_hidden`.

**Edge cases:** a source with 0 items shows `0`, not omission (honesty); `language: "und"` bucket rendered as "undetermined", never folded into English; disclosure block must pass P5-T2 publish validation — it carries no claims needing citations, verify the validator skips template blocks rather than weakening the gate.

## Compliance checklist (human verifies in mt-06)
- DPDP: no PII stored (re-run the paranoia PII check on telegram/play_store rows)
- Every report carries the coverage disclosure
- Telegram: public channels only

## Stop conditions
- Telethon auth requires a phone number you don't have → stop, hand to human (it's a one-time manual session setup).
- langid accuracy <90% on the fixture set → stop, report per-language confusion matrix.

## Manual test
`../manual-tests/mt-06-india.md`
