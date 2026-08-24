# WebSocket Protocol — Live Simulation Feed

**Version 0.1 · Source of truth for all real-time UI. Endpoint: `wss://<host>/api/v1/ws/simulations/{simulation_id}`**

## Connection & auth

- Phase 4+: client connects with `?ticket=<one-time-ticket>` where the ticket comes from `POST /api/v1/simulations/{sid}/ws-ticket` (JWT-authenticated REST call; ticket TTL 60s, single-use). Never put the Logto JWT in a WS URL — URLs end up in logs.
- Server sends `{"type":"hello","snapshot":{...}}` immediately: full current state (all runs, current rounds, last 50 posts) so a late joiner renders instantly.
- Reconnect: client sends `{"type":"resume","last_round":N}` — server replays rounds N+1..current, then continues live. Gap > buffer (500 rounds) → server sends fresh `hello` snapshot instead.

## Server → client events

```json
{ "type": "run_started",    "run_id": "uuid", "variant_id": "uuid", "ensemble_index": 0, "agent_count": 120 }
{ "type": "round",          "run_id": "uuid", "round": 7, "total_rounds": 18,
  "posts": [{ "post_id": "string", "agent_id": "string", "archetype": "string",
              "text": "string", "stance": "support|oppose|neutral",
              "reply_to": "post_id | null", "simulated": true }],
  "sentiment_tick": { "positive": 0.44, "neutral": 0.31, "negative": 0.25 } }
{ "type": "early_stop",     "run_id": "uuid", "round": 14, "reason": "flat_3_rounds" }
{ "type": "run_done",       "run_id": "uuid", "final_sentiment": {...} }
{ "type": "ensemble_update","variant_id": "uuid", "runs_done": 3, "agreement": 0.83 }
{ "type": "simulation_done","simulation_id": "uuid", "outcome": "verdict | no_consensus" }
{ "type": "budget_halt",    "project_id": "uuid", "spent": 24.97, "cap": 25.00 }
{ "type": "error",          "code": "STRING_ENUM", "message": "string" }
```

## Client → server messages

```json
{ "type": "resume", "last_round": 12 }
{ "type": "ask",    "run_id": "uuid", "agent_id": "string", "question": "string" }
```
`ask` answers arrive as `{"type":"ask_answer","simulated":true,"answer":"...","run_id":"...","round_context":[...]}`. Rate limit: 10 asks/min per user.

## Hard rules

1. **Every** post and ask_answer carries `simulated: true`. The UI must render the SIMULATED badge off this field — absence = render nothing (fail closed).
2. Event order within a run is by `round`; clients must buffer out-of-order arrivals (±2 rounds).
3. Server is a dumb proxy of MiroShark events + ensemble state from core-api. No simulation logic in the WS layer.
4. Backpressure: if client lags, server drops `round` events but never `run_done`/`simulation_done`/`budget_halt`; dropped ranges are flagged `{"type":"gap","from":N,"to":M}` and the client refetches via REST.
