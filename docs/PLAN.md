# aigent.community - LLM Token Sharing Marketplace

## Context

System allowing LLM subscription owners to share tokens with others via marketplace. Provider runs local agent or delegates API key; consumer browses pools on web UI, uses tokens in real-time chat sessions. Internal credit economy.

Built on existing `saas-on-cf` boilerplate (React 19 + TanStack Start, Hono on CF Workers, Drizzle + Neon Postgres, Better Auth).

## Architecture

```
Browser (Consumer/Provider)
  ↕ WebSocket + REST
TanStack Start SSR Worker ──service binding──► Hono API Worker
                                                ├─ SessionAgent (per session, agents-sdk)
                                                ├─ ProviderAgent (per provider, agents-sdk)
                                                ├─ KV (pool cache, online status)
                                                ├─ Queues (usage-log, credit-tx)
                                                └─ Neon PostgreSQL
Provider CLI Agent
  ↕ WebSocket
ProviderAgent
```

### Two proxy modes

1. **API key delegation** — provider stores encrypted key on platform, platform calls Claude API directly
2. **Local proxy** — provider runs CLI agent, platform routes requests via WebSocket to agent, agent calls Claude API locally

## Monorepo Structure (new/modified)

```
apps/
  cli/                          # NEW - provider CLI agent (Node.js)
    src/
      index.ts                  # entry (commander)
      commands/start.ts         # connect + proxy
      commands/status.ts        # check sessions/tokens
      proxy/ws-client.ts        # WS to platform
      proxy/llm-forwarder.ts    # forward to Claude API

  data-service/src/             # MODIFIED
    agents/
      session-agent.ts          # per-session Agent (extends Agent from agents-sdk)
      provider-agent.ts         # per-provider Agent for CLI bridge
    routes/
      pools.ts                  # CRUD token pools
      sessions.ts               # session lifecycle
      credits.ts                # balance + history
      marketplace.ts            # browse pools
      ws.ts                     # WS upgrade → DOs
    services/
      llm-proxy.ts              # Claude API streaming proxy
      token-meter.ts            # extract usage from API response
      credit-engine.ts          # reserve/settle/refund
    queue-handlers/
      usage-logger.ts           # batch insert usage
      credit-processor.ts       # settle credits

  user-application/src/routes/  # MODIFIED
    pools/index.tsx             # browse marketplace
    pools/$poolId.tsx           # pool detail
    session/$sessionId.tsx      # chat UI + token counter
    _auth/dashboard/provider.tsx
    _auth/dashboard/consumer.tsx
    _auth/settings/api-keys.tsx

packages/
  data-ops/src/drizzle/schema/ # MODIFIED - new tables
    providers.ts
    pools.ts
    sessions.ts
    credits.ts
    usage.ts
  data-ops/src/zod-schema/     # MODIFIED - new DTOs + WS types
    ws-messages.ts              # WebSocket message protocol types
    pool.ts                     # pool request/response schemas
    session.ts                  # session DTOs
    credit.ts                   # credit DTOs
```

## Database Schema (new tables)

### provider_config
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| user_id | text FK→user | |
| mode | enum | local_proxy \| api_key |
| llm_provider | text | "claude" default |
| encrypted_api_key | text | null if local_proxy |
| model_allowlist | jsonb | string[] |
| is_active | boolean | |
| last_seen_at | timestamp | heartbeat |
| reputation_score | real | 0.0–5.0, default 5.0 |
| total_sessions_served | int | |
| total_tokens_served | bigint | |

### token_pool
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | |
| provider_id | text FK→provider_config | |
| name, description | text | |
| status | enum | active/paused/depleted/revoked |
| available_from/to | text | "09:00"/"17:00" UTC |
| available_days | jsonb | [1,2,3,4,5] (0=Sun JS convention) |
| max_tokens_per_session | int | |
| max_tokens_per_day | int | |
| daily_tokens_used | int | resets daily |
| allowed_use_cases | jsonb | ["coding","writing"] |
| allowed_models | jsonb | ["claude-sonnet-4-20250514","claude-haiku-4-5-20251001"] |
| credits_per_input_k_token | int | |
| credits_per_output_k_token | int | |
| max_concurrent_sessions | int | |

### llm_session
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | |
| pool_id | text FK | |
| consumer_id, provider_id | text FK→user | |
| status | enum | pending/active/ending/completed/aborted/timeout |
| input/output_tokens_used | int | |
| model | text | claude model used |
| max_tokens_budget | int | output tokens only |
| credits_charged/reserved | int | |
| end_reason | enum | tokens_exhausted/time_window/user_ended/provider_revoked/provider_disconnected/error |

### credit_balance
| Column | Type |
|--------|------|
| user_id | text PK FK→user |
| available | int |
| reserved | int |

### credit_ledger
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | |
| user_id | text FK | |
| amount | int | +credit / -debit |
| type | enum | signup_bonus/provider_earning/consumer_spend/refund |
| session_id | text FK nullable | |
| balance_after | int | |

### usage_log
| Column | Type |
|--------|------|
| id | text PK |
| session_id | text FK |
| request_index | int |
| input/output_tokens | int |
| model | text |
| latency_ms | int |

## API Endpoints

```
# Provider config
POST/PATCH/GET  /api/provider/config
GET             /api/provider/stats

# Pools
POST/GET        /api/pools
PATCH/DELETE     /api/pools/:id
POST            /api/pools/:id/pause|resume

# Marketplace
GET             /api/marketplace?useCase=&minTokens=&sort=

# Sessions
POST            /api/sessions           { poolId, maxTokensBudget }
GET             /api/sessions/:id
POST            /api/sessions/:id/end
POST            /api/sessions/:id/summarize

# Credits
GET             /api/credits/balance
GET             /api/credits/history

# WebSocket upgrades (custom Hono WS routes with auth middleware)
GET             /api/ws/session/:sessionId    → SessionAgent
GET             /api/ws/provider              → ProviderAgent
```

## Agents (Cloudflare Agents SDK on Durable Objects)

Uses `agents-sdk` npm — framework on top of DOs. Same runtime/billing/limits, better DX.
Key features used: `this.state` auto-sync, multi-schedule, `@callable()` RPC, WS hibernation, `useAgentChat()` React hook.

### SessionAgent (1 per active session)

```typescript
class SessionAgent extends Agent<Env, SessionState> {
  // state auto-persisted + auto-synced to consumer WS
  // SessionState = { tokens, credits, history, budget, status }

  @callable()
  async streamResponse(message: string): Promise<void>

  @callable()
  async endSession(): Promise<void>

  @callable()
  async summarizeSession(): Promise<string>

  @callable()
  async exportSession(format: "md" | "json"): Promise<string>

  // Multi-schedule (replaces single alarm)
  // this.schedule("30m", "checkIdle")
  // this.schedule(poolCloseTime, "endByTimeWindow")
  // this.schedule("periodic", "1m", "syncCredits")

  onConnect(connection, ctx): void  // auth check, setup
  onMessage(connection, message): void  // chat messages
  onClose(connection, code, reason): void  // cleanup
}
```

- `this.state` with token counters auto-broadcasts to consumer WS → live token counter
- At 80%: `session_warning` via broadcast
- At 95%: second warning + prompt to save/summarize
- On end: settle credits via Queue, persist final state to DB
- **Conversation stored in Agent state only (ephemeral)** — UI must clearly warn user:
  - Banner on session start: "Conversation is NOT persisted. Save/export before ending."
  - Persistent indicator showing session is ephemeral
  - Before session ends: modal with save/compress/summarize options
  - Export formats: markdown, JSON

### ProviderAgent (1 per provider)

```typescript
class ProviderAgent extends Agent<Env, ProviderState> {
  // ProviderState = { online, activeRequests, stats }

  @callable()
  async forwardLlmRequest(request: LlmRequest): Promise<void>
  // streaming: ProviderAgent calls back SessionAgent via getAgentByName() per chunk

  @callable()
  async isOnline(): Promise<boolean>

  @callable()
  async getStats(): Promise<ProviderStats>

  onConnect(connection, ctx): void  // CLI agent connects
  onMessage(connection, message): void  // CLI agent responses
  onClose(connection, code, reason): void  // mark offline → KV
}
```

- Maintains WS to CLI agent
- Multiplexes requests from multiple SessionAgents via `@callable`
- Heartbeat via `this.schedule("periodic", "30s", "heartbeat")`
- Online status → KV

## Session Flow

1. Consumer `POST /api/sessions` → credits pre-authorized (reserved)
2. Consumer connects WS → SessionDO initialized
3. Consumer sends message → SessionAgent routes to LLM:
   - API key mode: direct Claude API call from Worker
   - Local proxy: `@callable` RPC to ProviderAgent → WS to CLI → Claude API → stream back
4. Response streams as `stream_chunk` messages
5. After each completion: `token_update` with live counters
6. 80% budget → warning; 95% → save/summarize prompt
7. Session end → credits settled (reserved released, actual charged), provider earns credits

## Credit Engine

```
credits = (input_tokens/1000) × creditsPerInputKToken
        + (output_tokens/1000) × creditsPerOutputKToken
```

- **Reserve** on session start (worst-case: all budget as output tokens × outputRate)
- **Settle** on session end in single `db.transaction()` (actual vs reserved, refund difference)
- **Earn** provider gets full credited amount (no platform fee in MVP)
- **Signup bonus**: 500 credits (~enough for a few medium sessions to try the platform)
- Credits are purely internal, non-cashable

### Provider disconnect mid-session
Simplest approach: **auto-refund unused reserved credits + end session**.
- SessionDO detects provider WS disconnect (via ProviderGatewayDO heartbeat timeout)
- Consumer gets `session_ended` message with reason "provider_disconnected"
- Save/export modal auto-triggered before connection fully closes
- Credits for unused tokens refunded immediately
- Provider reputation -0.1 per disconnect (incentivizes uptime)

## Wrangler Config Additions (wrangler.jsonc)

```jsonc
{
  "durable_objects": {
    "bindings": [
      { "name": "SESSION_AGENT", "class_name": "SessionAgent" },
      { "name": "PROVIDER_AGENT", "class_name": "ProviderAgent" }
    ]
  },
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["SessionAgent", "ProviderAgent"] }
  ],
  "kv_namespaces": [
    { "binding": "POOL_CACHE", "id": "<kv-namespace-id>" }
  ],
  "queues": {
    "producers": [
      { "binding": "USAGE_LOG_QUEUE", "queue": "usage-log" },
      { "binding": "CREDIT_TX_QUEUE", "queue": "credit-tx" }
    ],
    "consumers": [
      { "queue": "usage-log", "max_batch_size": 50, "max_batch_timeout": 10 },
      { "queue": "credit-tx", "max_batch_size": 10, "max_batch_timeout": 5 }
    ]
  }
}
```

## Security

- API keys encrypted AES-256-GCM before DB storage (ENCRYPTION_KEY as Worker secret). `api_key` mode blocked until Phase 5 encryption is implemented — Phase 1-4 support `local_proxy` mode only.
- Better Auth session required for all API routes
- WS upgrade validates auth before routing to DO
- Credit balance uses optimistic locking to prevent overspend
- CLI agent authenticates via long-lived API token (generated in settings)
- All inputs validated with Zod
- Local proxy mode: API key never leaves provider's machine

## Implementation Phases

### Phase 1 — Schema + Auth (foundation)
- Drizzle schema + migrations for all new tables
- Provider config CRUD
- Token pool CRUD
- Credit balance initialization on signup

### Phase 2 — Core Session (critical path)
- SessionAgent + ProviderAgent (agents-sdk, WS hibernation, @callable RPC)
- `useAgentChat()` React hook for consumer chat UI
- LLM proxy service (API key mode first)
- Token metering from Claude API `usage` field
- Credit engine (reserve/settle)
- Chat UI + token counter component

### Phase 3 — Marketplace + Dashboards
- Marketplace browse with KV caching
- Provider dashboard (stats, active sessions, earnings)
- Consumer dashboard (history, balance)
- Queue handlers for async usage logging + credit processing

### Phase 4 — CLI Agent + Local Proxy
- CLI scaffolding (commander + ws)
- ProviderAgent `@callable` methods for CLI communication
- WS client with reconnection + heartbeat
- LLM forwarding with streaming

### Phase 5 — Polish
- 80%/95% token warnings
- Session summarize/compress endpoint
- DO alarms for time-window enforcement
- Rate limiting, key encryption hardening

## Resolved Decisions

1. Credits purely internal, non-cashable
2. Provider disconnect → auto-refund + end session + reputation penalty
3. Conversation ephemeral in DO only (clear UI warnings)
4. Provider decides allowed models per pool (multi-model per pool)
5. Simple reputation: 0–5 score, -0.1 per disconnect, visible on marketplace
6. No platform fee in MVP
7. 500 credits signup bonus
8. No content moderation in MVP
9. `maxTokensBudget` = output tokens only (input unpredictable, output is what providers pay for)
10. Credit settlement always in single `db.transaction()` (reserve/settle/ledger/session update)
11. Reserve worst-case = all budget as output tokens × `creditsPerOutputKToken` (output more expensive)
12. `api_key` mode gated behind Phase 5 encryption. Phases 1-4 = `local_proxy` only
13. WS auth via custom Hono WS routes (not `routeAgentRequest`) to keep auth middleware consistent
14. `@callable()` does NOT support callback params (JSON-serialized RPC). DO-to-DO streaming uses reverse RPC: ProviderAgent calls `sessionAgent.handleProviderChunk(msg)` via `getAgentByName()`
15. `available_days` uses JS convention: 0=Sun, 1=Mon, ..., 6=Sat. Default `[1,2,3,4,5]` = Mon-Fri
16. DO migrations use `new_sqlite_classes` (not `new_classes`) for agents-sdk SQLite state
