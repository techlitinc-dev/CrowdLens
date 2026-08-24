# Report Blocks Contract — Report Studio

**Version 0.1 · A report = versioned, ordered array of typed blocks. Every figure in every block must resolve to evidence or the block fails validation.**

## Report document

```json
{
  "report_id": "uuid",
  "project_id": "uuid",
  "version": 3,
  "title": "string",
  "status": "draft | published | archived",
  "blocks": [ { "id": "uuid", "type": "<block_type>", "props": {...}, "evidence_refs": [...] } ]
}
```

`evidence_refs` entries are exactly one of:
```json
{ "kind": "real", "item_id": "collected_items.item_id" }
{ "kind": "sim",  "run_id": "uuid", "post_id": "string", "round": 7 }
```
Validation (server-side, on publish): every quantitative or quoted statement in `props` must have ≥1 evidence_ref; every ref must resolve (item exists / sim post exists). Failure → 422 with the offending block ids.

## Block types (closed set — no others allowed)

| type | props (required) | Evidence rule |
|---|---|---|
| `verdict_card` | `recommendation`, `confidence`, `agreement_score`, `variant_id` | from `verdicts` row only |
| `sentiment_timeline` | `series_id`, `annotations[]` | each annotation → real item ref |
| `sim_comparison` | `simulation_id`, `variant_ids[]` | bars link to branch replays (`run_id`s) |
| `persona_panel` | `panel_id` | quotes → real item refs |
| `objection_map` | `clusters[]: {label, quote_refs[]}` | mixed real/sim refs, each labeled |
| `kg_embed` | `entity_ids[]`, `time_window` | Neo4j entities from the project's runs |
| `cost_ledger` | `project_id` | computed from `cost_ledger` — read-only |
| `markdown` | `md` | free text, but claims inside still need evidence_refs |

## API (extends api-surface.md)

| Method | Path | Notes |
|---|---|---|
| POST | `/projects/{id}/reports` | compose from template (`vertical` param selects default block set) |
| GET/PUT | `/reports/{rid}` | PUT = new version row, never mutate |
| POST | `/reports/{rid}/publish` | runs validation + Langfuse claim-evidence eval; failures block publish |
| POST | `/reports/{rid}/share` | → `{share_url, expires_at}`; view-only, watermarked, ≤30d expiry |
| GET | `/share/{token}` | public, no auth; watermark = viewer IP + timestamp rendered in footer |
| POST | `/reports/{rid}/export` | `{format: "pdf|pptx"}` → async job → MinIO URL; white-label via workspace branding |
| POST | `/reports/{rid}/ask` | `{question}` → RAG over THIS project's data only; answer cites block ids + evidence_refs |

## Export rules

PDF/PPTX rendering happens in a worker (WeasyPrint for PDF, python-pptx for PPTX). Simulated content keeps its SIMULATED badge in exports. Cost ledger block always included — agencies may restyle but not remove it in the Agency tier.
