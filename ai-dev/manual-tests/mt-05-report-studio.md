# MT-05 — Report Studio

**Prereq:** MT-04 green; a project with a completed verdict (MT-02).

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | `POST /projects/<pid>/reports` (default template) → open `/projects/<pid>/reports/<rid>` | Report renders: verdict card, sentiment timeline, sim comparison, objection map, cost ledger | |
| 2 | Edit in Studio: reorder two blocks, delete one, add a markdown block with text | New version on save; old version still renders at its URL | |
| 3 | **Evidence gate (negative):** edit a markdown block to contain a claim, remove its evidence refs, try to publish | Publish blocked; the offending block is named in the UI | |
| 4 | Fix step 3, publish, then `POST /reports/<rid>/share` → open share URL in incognito | View-only rendering, no editor chrome; watermark with IP+timestamp in footer | |
| 5 | Create a share link with 1-minute expiry (dev override) → wait → reopen | `410 Gone` | |
| 6 | `POST /reports/<rid>/export` pdf + pptx (workspace branding set: logo + accent color) | Both files: branding applied, SIMULATED badges intact, cost ledger present; PPTX speaker notes carry evidence refs | |
| 7 | **Ask-the-report:** ask "What's the strongest objection to variant B?" → then ask "What did our competitor's Q3 earnings say?" (not in project data) | Q1: answer cites block ids + evidence refs that resolve. Q2: refused with "insufficient grounded data" — no hallucinated answer | |
| 8 | **Branch what-if:** from the sim comparison block, rerun with a tweaked price | New simulation launches under the same caps/ensemble rules; on completion, comparison block offers old-vs-new | |
| 9 | **Cross-tenant RAG probe:** as user B (no access), hit `/reports/<rid>/ask` directly | 403; embeddings from another project never surface | |

**Sign-off:** all PASS (especially 3, 7) → phase 6. Step 7 Q2 answering confidently is a launch-blocking FAIL.
