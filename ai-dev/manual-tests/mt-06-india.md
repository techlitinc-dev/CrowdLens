# MT-06 — India Data Pack & Multilingual

**Prereq:** MT-05 green. Telegram session set up (human, one-time). Test topic with real Indian discussion, e.g. a Bollywood release or an Indian app launch.

| # | Step | Expected | Result |
|---|---|---|---|
| 1 | Create project with `sources: ["reddit","youtube","telegram","play_store"]`, `region: "in"`, `languages: ["en","hi","mr"]` | Accepted; Grounding Console shows 4 source cards | |
| 2 | After collection: `SELECT source, count(*) FROM collected_items GROUP BY source` | telegram + play_store rows present (3 Telegram channels minimum) | |
| 3 | **Telegram scope check:** verify all telegram items come from public channels (spot-check 5 URLs open without login) | 5/5 public; no group data | |
| 4 | **PII sweep on new sources:** eyeball 50 telegram/play_store rows | No sender names, no reviewer profile links | |
| 5 | Run analysis; read the baseline report | Hindi/Marathi quotes appear in original script with EN glosses — not pre-translated; Hinglish rows have `language: "hi-Latn"` | |
| 6 | Render the report in Hindi (`language=hi`) | UI + narrative in Hindi (Devanagari renders correctly); quotes + citations unchanged from the EN version | |
| 7 | Grounding Console coverage panel | Per-source × language counts; explicit "no coverage: ShareChat, WhatsApp" disclosure; the same disclosure block appears inside the published report | |
| 8 | **PDF export in Hindi** | Devanagari embedded and readable in the PDF (not tofu boxes) | |
| 9 | Persona panel for this project | Archetypes carry `languages[]`; a tier-2/3 archetype exists if the data supports it; samples match the archetype's languages | |

**Sign-off:** all PASS → phase 7. Step 4 FAIL = stop everything (DPDP).
