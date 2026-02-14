# 003 - Marketplace + Dashboards (Phase 3)

## Overview

Phase 3 adds the consumer-facing marketplace, provider/consumer dashboards, KV caching for pool listings, and async queue handlers for usage logging and credit settlement. Builds on the schema, auth, session agent, and credit engine from Phases 1-2.

## Context

After Phase 2 delivers core session flow (SessionAgent, credit engine, chat UI), Phase 3 makes the platform usable end-to-end: consumers can discover pools, providers can track earnings, and async work (usage logging, credit processing) is offloaded to queues.

## Goals

- Browsable marketplace with filtering, sorting, pagination, KV-cached responses
- Provider dashboard: stats, active sessions, pool management, earnings
- Consumer dashboard: credit balance, session history, credit transaction history
- Queue consumers for batch usage logging and credit settlement
- All new endpoints behind Better Auth session middleware

## Non-Goals

- Session creation/chat UI (Phase 2)
- CLI agent / local proxy (Phase 4)
- Token warnings / session summarize (Phase 5)
- Full-text search on pools
- Real-time dashboard updates via WebSocket

---

## 1. Marketplace API

### Route: `apps/data-service/src/hono/handlers/marketplace-handlers.ts`

```
GET /api/marketplace
```

**Query params** (all optional):

| Param | Type | Default | Notes |
|-------|------|---------|-------|
| `useCase` | `string` | - | Filter: pool `allowed_use_cases` contains value |
| `model` | `string` | - | Filter: pool `allowed_models` contains value |
| `minTokens` | `number` | - | Filter: `max_tokens_per_session >= minTokens` |
| `sort` | `enum` | `reputation` | `reputation` \| `price_asc` \| `price_desc` \| `newest` |
| `limit` | `number` | `20` | Pagination (max 50) |
| `offset` | `number` | `0` | Pagination |

### Zod Schemas

File: `packages/data-ops/src/zod-schema/marketplace.ts`

```ts
import { z } from "zod"

// Request
export const MarketplaceSortSchema = z.enum([
  "reputation",
  "price_asc",
  "price_desc",
  "newest",
])

export const MarketplaceQuerySchema = z.object({
  useCase: z.string().optional(),
  model: z.string().optional(),
  minTokens: z.coerce.number().int().positive().optional(),
  sort: MarketplaceSortSchema.default("reputation"),
  limit: z.coerce.number().int().min(1).max(50).default(20),
  offset: z.coerce.number().int().min(0).default(0),
})

// Response
export const MarketplacePoolSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string().nullable(),
  status: z.string(),
  allowedUseCases: z.array(z.string()),
  allowedModels: z.array(z.string()),
  creditsPerInputKToken: z.number(),
  creditsPerOutputKToken: z.number(),
  maxTokensPerSession: z.number(),
  maxConcurrentSessions: z.number(),
  availableFrom: z.string().nullable(),
  availableTo: z.string().nullable(),
  availableDays: z.array(z.number()).nullable(),
  provider: z.object({
    id: z.string(),
    reputationScore: z.number(),
    totalSessionsServed: z.number(),
  }),
})

export const MarketplaceResponseSchema = z.object({
  data: z.array(MarketplacePoolSchema),
  pagination: z.object({
    total: z.number(),
    limit: z.number(),
    offset: z.number(),
    hasMore: z.boolean(),
  }),
})

export type MarketplaceQuery = z.infer<typeof MarketplaceQuerySchema>
export type MarketplacePool = z.infer<typeof MarketplacePoolSchema>
export type MarketplaceResponse = z.infer<typeof MarketplaceResponseSchema>
```

### Handler

File: `apps/data-service/src/hono/handlers/marketplace-handlers.ts`

```ts
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import { MarketplaceQuerySchema } from "@repo/data-ops/zod-schema/marketplace"
import { authMiddleware } from "../middleware/auth"
import * as marketplaceService from "../services/marketplace-service"

const marketplace = new Hono<{ Bindings: Env }>()

marketplace.get(
  "/",
  zValidator("query", MarketplaceQuerySchema),
  async (c) => {
    const query = c.req.valid("query")
    const result = await marketplaceService.getMarketplaceListings(
      query,
      c.env.POOL_CACHE
    )
    return c.json(result)
  }
)

export default marketplace
```

**No auth middleware on marketplace GET** -- public browse. Auth required only for actions (session creation, etc).

### Service

File: `apps/data-service/src/hono/services/marketplace-service.ts`

```ts
import type {
  MarketplaceQuery,
  MarketplaceResponse,
} from "@repo/data-ops/zod-schema/marketplace"
import {
  getMarketplaceListings as getListingsQuery,
} from "@repo/data-ops/queries/marketplace"

export async function getMarketplaceListings(
  query: MarketplaceQuery,
  kv: KVNamespace
): Promise<MarketplaceResponse> {
  const cacheKey = buildCacheKey(query)
  const cached = await kv.get(cacheKey, { type: "json" })
  if (cached) return cached as MarketplaceResponse

  const result = await getListingsQuery(query)

  await kv.put(cacheKey, JSON.stringify(result), {
    expirationTtl: 300, // 5 min
  })

  return result
}
```

### Query

File: `packages/data-ops/src/queries/marketplace.ts`

Joins `token_pool` + `provider_config` where `token_pool.status = 'active'`. Applies filters from query params. Sorting:

- `reputation`: `provider_config.reputation_score DESC`
- `price_asc`: `credits_per_output_k_token ASC`
- `price_desc`: `credits_per_output_k_token DESC`
- `newest`: `token_pool.created_at DESC`

For jsonb array filters (`useCase`, `model`): use Drizzle `sql` tagged template with `@>` operator.

```ts
import { sql, eq, and, gte, desc, asc } from "drizzle-orm"
import { getDb } from "../database/setup"
import { tokenPool } from "../drizzle/schema"
import { providerConfig } from "../drizzle/schema"
import type { MarketplaceQuery, MarketplaceResponse } from "../zod-schema/marketplace"

export async function getMarketplaceListings(
  query: MarketplaceQuery
): Promise<MarketplaceResponse> {
  const db = getDb()
  const conditions = [eq(tokenPool.status, "active")]

  if (query.useCase) {
    conditions.push(
      sql`${tokenPool.allowedUseCases} @> ${JSON.stringify([query.useCase])}::jsonb`
    )
  }
  if (query.model) {
    conditions.push(
      sql`${tokenPool.allowedModels} @> ${JSON.stringify([query.model])}::jsonb`
    )
  }
  if (query.minTokens) {
    conditions.push(gte(tokenPool.maxTokensPerSession, query.minTokens))
  }

  const orderMap = {
    reputation: desc(providerConfig.reputationScore),
    price_asc: asc(tokenPool.creditsPerOutputKToken),
    price_desc: desc(tokenPool.creditsPerOutputKToken),
    newest: desc(tokenPool.createdAt),
  } as const

  const [data, countResult] = await Promise.all([
    db
      .select({
        id: tokenPool.id,
        name: tokenPool.name,
        description: tokenPool.description,
        status: tokenPool.status,
        allowedUseCases: tokenPool.allowedUseCases,
        allowedModels: tokenPool.allowedModels,
        creditsPerInputKToken: tokenPool.creditsPerInputKToken,
        creditsPerOutputKToken: tokenPool.creditsPerOutputKToken,
        maxTokensPerSession: tokenPool.maxTokensPerSession,
        maxConcurrentSessions: tokenPool.maxConcurrentSessions,
        availableFrom: tokenPool.availableFrom,
        availableTo: tokenPool.availableTo,
        availableDays: tokenPool.availableDays,
        provider: {
          id: providerConfig.id,
          reputationScore: providerConfig.reputationScore,
          totalSessionsServed: providerConfig.totalSessionsServed,
        },
      })
      .from(tokenPool)
      .innerJoin(providerConfig, eq(tokenPool.providerId, providerConfig.id))
      .where(and(...conditions))
      .orderBy(orderMap[query.sort])
      .limit(query.limit)
      .offset(query.offset),
    db
      .select({ total: count() })
      .from(tokenPool)
      .innerJoin(providerConfig, eq(tokenPool.providerId, providerConfig.id))
      .where(and(...conditions)),
  ])

  const total = countResult[0]?.total ?? 0
  return {
    data,
    pagination: {
      total,
      limit: query.limit,
      offset: query.offset,
      hasMore: query.offset + data.length < total,
    },
  }
}
```

### Pool Detail

```
GET /api/marketplace/:poolId
```

Returns single `MarketplacePool` with full provider info. Cached separately in KV.

---

## 2. Credits API

### Route: `apps/data-service/src/hono/handlers/credits-handlers.ts`

Both endpoints require auth (Better Auth session). User ID extracted from session.

#### GET /api/credits/balance

Response:

```ts
export const CreditBalanceResponseSchema = z.object({
  available: z.number(),
  reserved: z.number(),
})

export type CreditBalanceResponse = z.infer<typeof CreditBalanceResponseSchema>
```

#### GET /api/credits/history

Query params:

| Param | Type | Default |
|-------|------|---------|
| `type` | `enum` | - (all) |
| `limit` | `number` | `20` |
| `offset` | `number` | `0` |

Response:

```ts
export const CreditLedgerTypeSchema = z.enum([
  "signup_bonus",
  "provider_earning",
  "consumer_spend",
  "refund",
])

export const CreditLedgerEntrySchema = z.object({
  id: z.string(),
  amount: z.number(),
  type: CreditLedgerTypeSchema,
  sessionId: z.string().nullable(),
  balanceAfter: z.number(),
  createdAt: z.string().datetime(),
})

export const CreditHistoryQuerySchema = z.object({
  type: CreditLedgerTypeSchema.optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  offset: z.coerce.number().int().min(0).default(0),
})

export const CreditHistoryResponseSchema = z.object({
  data: z.array(CreditLedgerEntrySchema),
  pagination: z.object({
    total: z.number(),
    limit: z.number(),
    offset: z.number(),
    hasMore: z.boolean(),
  }),
})

export type CreditHistoryQuery = z.infer<typeof CreditHistoryQuerySchema>
export type CreditLedgerEntry = z.infer<typeof CreditLedgerEntrySchema>
export type CreditHistoryResponse = z.infer<typeof CreditHistoryResponseSchema>
```

### Handler

```ts
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import {
  CreditHistoryQuerySchema,
} from "@repo/data-ops/zod-schema/credit"
import { sessionMiddleware } from "../middleware/session"
import * as creditsService from "../services/credits-service"

const credits = new Hono<{ Bindings: Env }>()

credits.use("*", sessionMiddleware)

credits.get("/balance", async (c) => {
  const userId = c.get("userId")
  return c.json(await creditsService.getBalance(userId))
})

credits.get(
  "/history",
  zValidator("query", CreditHistoryQuerySchema),
  async (c) => {
    const userId = c.get("userId")
    const query = c.req.valid("query")
    return c.json(await creditsService.getHistory(userId, query))
  }
)

export default credits
```

### Service

File: `apps/data-service/src/hono/services/credits-service.ts`

- `getBalance(userId)` -- queries `credit_balance` table by PK
- `getHistory(userId, query)` -- queries `credit_ledger` filtered by `user_id`, optional `type`, ordered `created_at DESC`, paginated

Both delegate to `@repo/data-ops/queries/credits.ts`.

---

## 3. Provider Stats API

### Route: `apps/data-service/src/hono/handlers/provider-handlers.ts`

```
GET /api/provider/stats
```

Auth required. Provider ID resolved from session user.

Response:

```ts
export const ProviderStatsResponseSchema = z.object({
  totalSessionsServed: z.number(),
  totalTokensServed: z.number(),
  totalEarnings: z.number(),
  activeSessionsCount: z.number(),
  reputationScore: z.number(),
  pools: z.object({
    total: z.number(),
    active: z.number(),
    paused: z.number(),
  }),
})

export type ProviderStatsResponse = z.infer<typeof ProviderStatsResponseSchema>
```

### Service

File: `apps/data-service/src/hono/services/provider-service.ts`

`getProviderStats(userId)`:

1. Query `provider_config` for `totalSessionsServed`, `totalTokensServed`, `reputationScore`
2. Query `credit_ledger` SUM where `type = 'provider_earning'` and `user_id`
3. Query `llm_session` COUNT where `provider_id = userId` and `status = 'active'`
4. Query `token_pool` COUNT grouped by status where `provider_id`

All via `Promise.all()` for parallel execution.

---

## 4. Queue Handlers

### 4.1 Usage Logger

File: `apps/data-service/src/queues/usage-logger.ts`

**Purpose:** Batch insert `usage_log` records from `USAGE_LOG_QUEUE`. Produced by SessionAgent after each LLM response completes.

**Queue message schema:**

```ts
export const UsageLogMessageSchema = z.object({
  sessionId: z.string(),
  requestIndex: z.number().int(),
  inputTokens: z.number().int(),
  outputTokens: z.number().int(),
  model: z.string(),
  latencyMs: z.number().int(),
  timestamp: z.number(),
})

export type UsageLogMessage = z.infer<typeof UsageLogMessageSchema>
```

**Consumer logic:**

```ts
export async function handleUsageLogQueue(
  batch: MessageBatch<UsageLogMessage>,
  env: Env
): Promise<void> {
  const validRecords: UsageLogMessage[] = []

  for (const msg of batch.messages) {
    const result = UsageLogMessageSchema.safeParse(msg.body)
    if (!result.success) {
      console.error("Invalid usage log message:", msg.id, result.error)
      msg.ack() // drop invalid, don't retry
      continue
    }
    validRecords.push(result.data)
  }

  if (validRecords.length === 0) {
    batch.ackAll()
    return
  }

  try {
    await batchInsertUsageLogs(validRecords)
    batch.ackAll()
  } catch (error) {
    console.error("Usage log batch insert failed:", error)
    // all messages retry (auto)
    batch.retryAll()
  }
}
```

**data-ops query** (`packages/data-ops/src/queries/usage.ts`):

```ts
export async function batchInsertUsageLogs(records: UsageLogMessage[]) {
  const db = getDb()
  await db.insert(usageLog).values(
    records.map((r) => ({
      id: generateUlid(),
      sessionId: r.sessionId,
      requestIndex: r.requestIndex,
      inputTokens: r.inputTokens,
      outputTokens: r.outputTokens,
      model: r.model,
      latencyMs: r.latencyMs,
    }))
  )
}
```

### 4.2 Credit Processor

File: `apps/data-service/src/queues/credit-processor.ts`

**Purpose:** Settle credit transactions from `CREDIT_TX_QUEUE`. Produced by SessionAgent on session end.

**Queue message schema:**

```ts
export const CreditTxMessageSchema = z.object({
  type: z.enum(["settle", "refund"]),
  sessionId: z.string(),
  consumerId: z.string(),
  providerId: z.string(),
  inputTokensUsed: z.number().int(),
  outputTokensUsed: z.number().int(),
  creditsReserved: z.number().int(),
  creditsPerInputKToken: z.number().int(),
  creditsPerOutputKToken: z.number().int(),
  timestamp: z.number(),
})

export type CreditTxMessage = z.infer<typeof CreditTxMessageSchema>
```

**Consumer logic:**

```ts
export async function handleCreditTxQueue(
  batch: MessageBatch<CreditTxMessage>,
  env: Env
): Promise<void> {
  for (const msg of batch.messages) {
    const result = CreditTxMessageSchema.safeParse(msg.body)
    if (!result.success) {
      console.error("Invalid credit tx message:", msg.id, result.error)
      msg.ack()
      continue
    }

    try {
      await settleCreditTransaction(result.data)
      msg.ack()
    } catch (error) {
      console.error("Credit settlement failed:", msg.id, error)
      msg.retry()
    }
  }
}
```

**Per-message processing** (not batch) -- credit settlement must be atomic per session. Each message processed in its own transaction:

1. Calculate actual cost: `Math.ceil((inputTokens/1000) * creditsPerInputKToken + (outputTokens/1000) * creditsPerOutputKToken)` (always round up to integer credits)
2. Release reserved credits from consumer (`credit_balance.reserved -= creditsReserved`)
3. Debit actual cost from consumer (`credit_balance.available -= actualCost`)
4. Credit provider (`credit_balance.available += actualCost`)
5. Refund difference if `creditsReserved > actualCost` (add back to consumer available)
6. Insert `credit_ledger` entries for consumer (type: `consumer_spend`) and provider (type: `provider_earning`)
7. Update `llm_session.credits_charged = actualCost`

All within a single DB transaction using Drizzle `db.transaction()`.

### 4.3 Error Handling & Retry

| Queue | Strategy | Rationale |
|-------|----------|-----------|
| `usage-log` | Batch ack/retry. Invalid messages acked (dropped). DB failure retries all. | Usage logs are append-only, safe to batch. |
| `credit-tx` | Per-message ack/retry. Invalid acked. Settlement failure retries individual message. | Credit mutations must be atomic per session. |

CF Queues retry up to `max_retries` (default 3) with exponential backoff. After exhaustion, messages go to DLQ if configured.

### 4.4 Wrangler Queue Config

Add to `apps/data-service/wrangler.jsonc` per environment:

```jsonc
"queues": {
  "producers": [
    { "binding": "USAGE_LOG_QUEUE", "queue": "usage-log-dev" },
    { "binding": "CREDIT_TX_QUEUE", "queue": "credit-tx-dev" }
  ],
  "consumers": [
    {
      "queue": "usage-log-dev",
      "max_batch_size": 50,
      "max_batch_timeout": 10,
      "max_retries": 3,
      "dead_letter_queue": "usage-log-dlq-dev"
    },
    {
      "queue": "credit-tx-dev",
      "max_batch_size": 10,
      "max_batch_timeout": 5,
      "max_retries": 3,
      "dead_letter_queue": "credit-tx-dlq-dev"
    }
  ]
}
```

### 4.5 Worker Entry Integration

Update `apps/data-service/src/index.ts`:

```ts
import { handleUsageLogQueue } from "./queues/usage-logger"
import { handleCreditTxQueue } from "./queues/credit-processor"

// In DataService class:
async queue(batch: MessageBatch<unknown>) {
  switch (batch.queue) {
    case "usage-log-dev":
    case "usage-log-staging":
    case "usage-log-prod":
      await handleUsageLogQueue(
        batch as MessageBatch<UsageLogMessage>,
        this.env
      )
      break
    case "credit-tx-dev":
    case "credit-tx-staging":
    case "credit-tx-prod":
      await handleCreditTxQueue(
        batch as MessageBatch<CreditTxMessage>,
        this.env
      )
      break
    default:
      console.error("Unknown queue:", batch.queue)
  }
}
```

Update `Env` interface via `pnpm run cf-typegen` after wrangler.jsonc changes:

```ts
interface Env {
  // ... existing
  POOL_CACHE: KVNamespace
  USAGE_LOG_QUEUE: Queue
  CREDIT_TX_QUEUE: Queue
}
```

---

## 5. KV Caching Layer

### Binding

Name: `POOL_CACHE` (KVNamespace)

### What Gets Cached

| Data | Key Pattern | TTL |
|------|-------------|-----|
| Marketplace listing pages | `mp:v1:{queryHash}` | 300s (5 min) |
| Pool detail | `pool:v1:{poolId}` | 600s (10 min) |

`queryHash` = deterministic hash of sorted query params (using `JSON.stringify` of sorted entries).

### Key Construction

```ts
function buildCacheKey(query: MarketplaceQuery): string {
  const sorted = Object.entries(query)
    .filter(([, v]) => v !== undefined)
    .sort(([a], [b]) => a.localeCompare(b))
  const hash = JSON.stringify(sorted)
  return `mp:v1:${hash}`
}
```

### Cache Invalidation

Triggered when a pool is updated, paused, resumed, or deleted.

In `apps/data-service/src/hono/services/pool-service.ts` (Phase 1 code), after any pool mutation:

```ts
async function invalidatePoolCache(
  kv: KVNamespace,
  poolId: string
): Promise<void> {
  // Delete specific pool detail cache
  await kv.delete(`pool:v1:${poolId}`)

  // Delete all marketplace listing caches (paginated — kv.list returns max 1000 per call)
  let cursor: string | undefined
  do {
    const result = await kv.list({ prefix: "mp:v1:", cursor })
    if (result.keys.length > 0) {
      await Promise.all(result.keys.map((k) => kv.delete(k.name)))
    }
    cursor = result.list_complete ? undefined : result.cursor
  } while (cursor)
}
```

**Trade-off:** Prefix-based purge is O(n) on cached keys. Acceptable at MVP scale (low hundreds of cached query variants). If pool mutations are rare relative to reads, this is fine. At scale, switch to versioned cache keys or pub/sub invalidation.

### TTL Strategy

- 5 min for marketplace listings: balances freshness with cache hit rate. Consumers see near-real-time availability.
- 10 min for pool detail: changes less frequently accessed, slightly stale OK.
- No cache on credits/stats endpoints: these are user-specific, low volume, must be current.

---

## 6. Frontend Pages

All frontend pages use the **binding pattern**: `createServerFn()` --> `DATA_SERVICE.fetch()` --> data-service API.

Server functions go in `apps/user-application/src/core/functions/`.

### 6.1 Marketplace Browse -- `pools/index.tsx`

Route: `/pools` (public, no `_auth` prefix)

File: `apps/user-application/src/routes/pools/index.tsx`

#### Server Function

File: `apps/user-application/src/core/functions/marketplace/browse.ts`

```ts
import { createServerFn } from "@tanstack/react-start"
import { env } from "cloudflare:workers"
import {
  MarketplaceQuerySchema,
  MarketplaceResponseSchema,
  type MarketplaceQuery,
  type MarketplaceResponse,
} from "@repo/data-ops/zod-schema/marketplace"

export const getMarketplaceListings = createServerFn()
  .inputValidator((data: MarketplaceQuery) =>
    MarketplaceQuerySchema.parse(data)
  )
  .handler(async (ctx): Promise<MarketplaceResponse> => {
    const params = new URLSearchParams()
    if (ctx.data.useCase) params.set("useCase", ctx.data.useCase)
    if (ctx.data.model) params.set("model", ctx.data.model)
    if (ctx.data.minTokens) params.set("minTokens", String(ctx.data.minTokens))
    params.set("sort", ctx.data.sort)
    params.set("limit", String(ctx.data.limit))
    params.set("offset", String(ctx.data.offset))

    const response = await env.DATA_SERVICE.fetch(
      new Request(`https://data-service/api/marketplace?${params}`)
    )
    if (!response.ok) throw new Error("Failed to fetch marketplace listings")
    return MarketplaceResponseSchema.parse(await response.json())
  })
```

#### Query Options

File: `apps/user-application/src/lib/query-keys.ts` (extend existing)

```ts
export const marketplaceKeys = {
  all: ["marketplace"] as const,
  list: (query: MarketplaceQuery) =>
    [...marketplaceKeys.all, "list", query] as const,
  detail: (poolId: string) =>
    [...marketplaceKeys.all, "detail", poolId] as const,
}

export const marketplaceListQueryOptions = (query: MarketplaceQuery) =>
  queryOptions({
    queryKey: marketplaceKeys.list(query),
    queryFn: () => getMarketplaceListings({ data: query }),
    placeholderData: (prev) => prev,
    staleTime: 1000 * 60, // 1 min client-side
  })
```

#### Route Definition

```tsx
import { createFileRoute, useNavigate } from "@tanstack/react-router"
import { useQuery } from "@tanstack/react-query"
import { MarketplaceQuerySchema } from "@repo/data-ops/zod-schema/marketplace"
import { marketplaceListQueryOptions } from "@/lib/query-keys"

export const Route = createFileRoute("/pools/")({
  validateSearch: MarketplaceQuerySchema,
  loaderDeps: ({ search }) => search,
  loader: async ({ context, deps }) => {
    await context.queryClient.ensureQueryData(
      marketplaceListQueryOptions(deps)
    )
  },
  component: MarketplacePage,
})
```

#### Component Structure

```
MarketplacePage
  |-- MarketplaceFilters       (useCase, model, minTokens, sort selects)
  |-- PoolCardGrid
  |     |-- PoolCard[]         (name, provider rep, price, models, status)
  |-- PaginationControls       (prev/next, offset-based)
```

**Zero state:** "No pools available matching your filters. Try broadening your search."

**MarketplaceFilters:** Uses `validateSearch` + `useNavigate` for URL-driven state (per rules: no `useState` for URL-driven state). Each filter change navigates with updated search params, resetting offset to 0.

```tsx
function MarketplaceFilters() {
  const search = Route.useSearch()
  const navigate = useNavigate()

  return (
    <div className="flex flex-wrap gap-4">
      <Select
        value={search.useCase ?? ""}
        onValueChange={(v) =>
          navigate({
            to: "/pools",
            search: { ...search, useCase: v || undefined, offset: 0 },
          })
        }
      >
        {/* coding, writing, analysis, general */}
      </Select>
      {/* model select, minTokens input, sort select */}
    </div>
  )
}
```

**PoolCard:**

```tsx
interface PoolCardProps {
  pool: MarketplacePool
}

export function PoolCard({ pool }: PoolCardProps) {
  return (
    <Link to="/pools/$poolId" params={{ poolId: pool.id }}>
      <Card className="hover:border-primary transition-colors">
        <CardHeader>
          <div className="flex items-center justify-between">
            <CardTitle className="text-lg">{pool.name}</CardTitle>
            <Badge variant={pool.status === "active" ? "default" : "secondary"}>
              {pool.status}
            </Badge>
          </div>
          <CardDescription>{pool.description}</CardDescription>
        </CardHeader>
        <CardContent className="space-y-3">
          <div className="flex items-center gap-2">
            <Star className="h-4 w-4 text-yellow-500" />
            <span className="text-sm text-foreground">
              {pool.provider.reputationScore.toFixed(1)}
            </span>
            <span className="text-xs text-muted-foreground">
              ({pool.provider.totalSessionsServed} sessions)
            </span>
          </div>
          <div className="flex flex-wrap gap-1">
            {pool.allowedModels.map((m) => (
              <Badge key={m} variant="outline" className="text-xs">
                {m}
              </Badge>
            ))}
          </div>
          <div className="text-sm text-muted-foreground">
            {pool.creditsPerOutputKToken} credits/1k output tokens
          </div>
        </CardContent>
      </Card>
    </Link>
  )
}
```

### 6.2 Pool Detail -- `pools/$poolId.tsx`

Route: `/pools/:poolId` (public)

File: `apps/user-application/src/routes/pools/$poolId.tsx`

#### Server Function

File: `apps/user-application/src/core/functions/marketplace/detail.ts`

```ts
export const getPoolDetail = createServerFn()
  .inputValidator((data: { poolId: string }) =>
    z.object({ poolId: z.string().min(1) }).parse(data)
  )
  .handler(async (ctx): Promise<MarketplacePool | null> => {
    const response = await env.DATA_SERVICE.fetch(
      new Request(`https://data-service/api/marketplace/${ctx.data.poolId}`)
    )
    if (response.status === 404) return null
    if (!response.ok) throw new Error("Failed to fetch pool")
    return MarketplacePoolSchema.parse(await response.json())
  })
```

#### Route

```tsx
export const Route = createFileRoute("/pools/$poolId")({
  loader: async ({ context, params }) => {
    await context.queryClient.ensureQueryData(
      poolDetailQueryOptions(params.poolId)
    )
  },
  component: PoolDetailPage,
})
```

#### Component Structure

```
PoolDetailPage
  |-- PoolHeader          (name, status badge, provider reputation)
  |-- PoolDetails         (models, use cases, token limits, pricing)
  |-- AvailabilitySchedule (days + time window display)
  |-- StartSessionButton  (Link to session creation, requires auth)
```

**AvailabilitySchedule** renders `availableDays` as day chips (Mon-Sun) and `availableFrom`/`availableTo` as time range.

**StartSessionButton** links to session creation flow (Phase 2 route). If user not authed, redirects to signin.

### 6.3 Provider Dashboard -- `_auth/dashboard/provider.tsx`

Route: `/dashboard/provider` (auth required via `_auth` layout)

File: `apps/user-application/src/routes/_auth/dashboard/provider.tsx`

#### Server Functions

File: `apps/user-application/src/core/functions/provider/stats.ts`

```ts
// Service bindings are same-zone trusted — no API token needed.
// Forward the user's session cookie so data-service sessionMiddleware resolves the user.
export const getProviderStats = createServerFn()
  .handler(async ({ request }): Promise<ProviderStatsResponse> => {
    const response = await env.DATA_SERVICE.fetch(
      new Request("https://data-service/api/provider/stats", {
        headers: {
          Cookie: request.headers.get("Cookie") ?? "",
        },
      })
    )
    if (!response.ok) throw new Error("Failed to fetch provider stats")
    return ProviderStatsResponseSchema.parse(await response.json())
  })
```

Additional server functions needed:

- `getProviderPools()` -- GET /api/pools (filtered by current user's provider)
- `getProviderActiveSessions()` -- GET /api/sessions?status=active&role=provider
- `getProviderEarningsHistory()` -- GET /api/credits/history?type=provider_earning

#### Route

```tsx
export const Route = createFileRoute("/_auth/dashboard/provider")({
  loader: async ({ context }) => {
    await Promise.all([
      context.queryClient.ensureQueryData(providerStatsQueryOptions()),
      context.queryClient.ensureQueryData(providerPoolsQueryOptions()),
    ])
  },
  component: ProviderDashboard,
})
```

#### Component Structure

```
ProviderDashboard
  |-- StatsOverview
  |     |-- StatCard (Total Sessions)
  |     |-- StatCard (Total Tokens)
  |     |-- StatCard (Total Earnings)
  |     |-- StatCard (Reputation)
  |-- ActiveSessionsList     (table: session ID, consumer, tokens used, duration)
  |-- PoolManagement
  |     |-- PoolRow[]        (name, status, pause/resume button)
  |-- EarningsTable          (credit_ledger entries, type=provider_earning)
```

**StatCard:**

```tsx
interface StatCardProps {
  label: string
  value: string | number
  icon: React.ComponentType<{ className?: string }>
}

export function StatCard({ label, value, icon: Icon }: StatCardProps) {
  return (
    <Card>
      <CardContent className="flex items-center gap-4 p-6">
        <div className="rounded-full bg-primary/10 p-3">
          <Icon className="h-6 w-6 text-primary" />
        </div>
        <div>
          <p className="text-sm text-muted-foreground">{label}</p>
          <p className="text-2xl font-bold text-foreground">{value}</p>
        </div>
      </CardContent>
    </Card>
  )
}
```

**Zero states:**
- No pools: "You haven't created any token pools yet. Create your first pool to start earning."
- No active sessions: "No active sessions right now."
- No earnings: "No earnings yet. Sessions will appear here once consumers use your pools."

**Pool pause/resume** uses `useMutation` calling a server function that POSTs to `/api/pools/:id/pause` or `/api/pools/:id/resume`, then invalidates `providerPoolsQueryOptions`.

### 6.4 Consumer Dashboard -- `_auth/dashboard/consumer.tsx`

Route: `/dashboard/consumer` (auth required)

File: `apps/user-application/src/routes/_auth/dashboard/consumer.tsx`

#### Server Functions

File: `apps/user-application/src/core/functions/consumer/dashboard.ts`

- `getCreditBalance()` -- GET /api/credits/balance
- `getSessionHistory(params)` -- GET /api/sessions?role=consumer&status=completed (paginated)
- `getCreditHistory(params)` -- GET /api/credits/history

#### Route

```tsx
export const Route = createFileRoute("/_auth/dashboard/consumer")({
  loader: async ({ context }) => {
    await Promise.all([
      context.queryClient.ensureQueryData(creditBalanceQueryOptions()),
      context.queryClient.ensureQueryData(
        sessionHistoryQueryOptions({ limit: 10, offset: 0 })
      ),
    ])
  },
  component: ConsumerDashboard,
})
```

#### Component Structure

```
ConsumerDashboard
  |-- CreditBalanceCard      (available, reserved amounts)
  |-- SessionHistoryTable    (pool name, tokens used, cost, duration, date)
  |     |-- PaginationControls
  |-- CreditTransactionTable (amount, type badge, session link, date)
  |     |-- PaginationControls
```

**CreditBalanceCard:**

```tsx
function CreditBalanceCard({ balance }: { balance: CreditBalanceResponse }) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>Credit Balance</CardTitle>
      </CardHeader>
      <CardContent className="grid grid-cols-2 gap-6">
        <div>
          <p className="text-sm text-muted-foreground">Available</p>
          <p className="text-3xl font-bold text-foreground">
            {balance.available}
          </p>
        </div>
        <div>
          <p className="text-sm text-muted-foreground">Reserved</p>
          <p className="text-3xl font-bold text-muted-foreground">
            {balance.reserved}
          </p>
        </div>
      </CardContent>
    </Card>
  )
}
```

**Zero states:**
- No sessions: "No sessions yet. Browse the marketplace to find a provider and start chatting."
- No credit transactions: "No transactions yet. Your credit history will appear here after your first session."

**SessionHistoryTable** displays completed sessions with:
- Pool name (linked to pool detail)
- Input/output tokens used
- Credits charged
- Session duration (calculated from `created_at` / `ended_at`)
- Date

Uses `validateSearch` for pagination params on URL.

---

## 7. Session Auth Middleware

A new middleware for data-service that extracts the authenticated user from Better Auth session cookie/token (forwarded via service binding).

File: `apps/data-service/src/hono/middleware/session.ts`

```ts
import { createMiddleware } from "hono/factory"
import { HTTPException } from "hono/http-exception"

export const sessionMiddleware = createMiddleware<{ Bindings: Env }>(
  async (c, next) => {
    // Extract session from Better Auth cookie/header forwarded from user-application
    const session = await validateSession(c.req.raw, c.env)
    if (!session) {
      throw new HTTPException(401, { message: "Authentication required" })
    }
    c.set("userId", session.userId)
    await next()
  }
)
```

Implementation depends on Phase 1 auth setup. The key point: session middleware sets `userId` on Hono context, consumed by credits/provider handlers.

---

## 8. Navigation Updates

Update `apps/user-application/src/components/layout/sidebar.tsx` navigation items:

```ts
const navigationItems: NavigationItem[] = [
  { name: "Home", icon: Home, href: "/" },
  { name: "Marketplace", icon: Store, href: "/pools" },
  { name: "Provider", icon: Server, href: "/dashboard/provider" },
  { name: "Consumer", icon: User, href: "/dashboard/consumer" },
]
```

---

## 9. Hono Route Registration

Update `apps/data-service/src/hono/app.ts`:

```ts
import marketplace from "./handlers/marketplace-handlers"
import credits from "./handlers/credits-handlers"
import provider from "./handlers/provider-handlers"

App.route("/api/marketplace", marketplace)
App.route("/api/credits", credits)
App.route("/api/provider", provider)
```

---

## 10. New Files Summary

### data-ops

| File | Purpose |
|------|---------|
| `src/zod-schema/marketplace.ts` | Marketplace query/response schemas |
| `src/zod-schema/credit.ts` | Credit balance/history/ledger schemas |
| `src/zod-schema/provider-stats.ts` | Provider stats response schema |
| `src/zod-schema/queue-messages.ts` | UsageLogMessage, CreditTxMessage schemas |
| `src/queries/marketplace.ts` | Marketplace listing query with filters/sort |
| `src/queries/credits.ts` | Balance + ledger history queries |
| `src/queries/provider-stats.ts` | Aggregated provider stats query |
| `src/queries/usage.ts` | Batch insert usage logs |
| `src/queries/credit-settlement.ts` | Atomic credit settlement transaction |

### data-service

| File | Purpose |
|------|---------|
| `src/hono/handlers/marketplace-handlers.ts` | GET /api/marketplace, GET /api/marketplace/:poolId |
| `src/hono/handlers/credits-handlers.ts` | GET /api/credits/balance, GET /api/credits/history |
| `src/hono/handlers/provider-handlers.ts` | GET /api/provider/stats |
| `src/hono/services/marketplace-service.ts` | KV-cached marketplace queries |
| `src/hono/services/credits-service.ts` | Credit balance + history |
| `src/hono/services/provider-service.ts` | Provider stats aggregation |
| `src/hono/middleware/session.ts` | Better Auth session validation |
| `src/queues/usage-logger.ts` | Queue consumer: batch usage log insert |
| `src/queues/credit-processor.ts` | Queue consumer: credit settlement |

### user-application

| File | Purpose |
|------|---------|
| `src/routes/pools/index.tsx` | Marketplace browse page |
| `src/routes/pools/$poolId.tsx` | Pool detail page |
| `src/routes/_auth/dashboard/provider.tsx` | Provider dashboard |
| `src/routes/_auth/dashboard/consumer.tsx` | Consumer dashboard |
| `src/core/functions/marketplace/browse.ts` | Server fn: marketplace listings |
| `src/core/functions/marketplace/detail.ts` | Server fn: pool detail |
| `src/core/functions/provider/stats.ts` | Server fn: provider stats |
| `src/core/functions/provider/pools.ts` | Server fn: provider pool management |
| `src/core/functions/consumer/dashboard.ts` | Server fn: credit balance, history |
| `src/components/marketplace/pool-card.tsx` | Pool card component |
| `src/components/marketplace/marketplace-filters.tsx` | Filter bar |
| `src/components/dashboard/stat-card.tsx` | Reusable stat card |
| `src/components/dashboard/credit-balance-card.tsx` | Credit display |

---

## 11. Verification Checklist

### Marketplace API

- [ ] `GET /api/marketplace` returns active pools sorted by reputation (default)
- [ ] `GET /api/marketplace?useCase=coding` filters to pools with coding use case
- [ ] `GET /api/marketplace?model=claude-sonnet-4-20250514` filters to pools with that model
- [ ] `GET /api/marketplace?minTokens=5000` excludes pools with maxTokensPerSession < 5000
- [ ] `GET /api/marketplace?sort=price_asc` sorts by creditsPerOutputKToken ascending
- [ ] `GET /api/marketplace?limit=5&offset=5` returns second page of 5
- [ ] Response includes `pagination.hasMore` = true when more pages exist
- [ ] Response includes nested `provider.reputationScore` per pool
- [ ] Paused/depleted/revoked pools excluded from results

### KV Caching

- [ ] First marketplace request populates KV cache (verify via `POOL_CACHE.get`)
- [ ] Second identical request returns cached data (no DB hit)
- [ ] Different query params produce different cache keys
- [ ] Cache expires after 5 minutes (TTL)
- [ ] Pool update triggers `invalidatePoolCache` and purges marketplace cache keys
- [ ] Pool pause triggers cache invalidation
- [ ] Pool detail cached separately with 10 min TTL

### Credits API

- [ ] `GET /api/credits/balance` returns 401 without auth
- [ ] `GET /api/credits/balance` returns `{ available, reserved }` for authed user
- [ ] `GET /api/credits/history` returns paginated ledger entries newest-first
- [ ] `GET /api/credits/history?type=provider_earning` filters to earnings only
- [ ] Credit balance matches sum of all ledger entries for that user

### Provider Stats

- [ ] `GET /api/provider/stats` returns 401 without auth
- [ ] `totalSessionsServed` matches provider_config value
- [ ] `totalEarnings` matches SUM of credit_ledger where type=provider_earning
- [ ] `activeSessionsCount` matches COUNT of llm_session where status=active
- [ ] `pools.active` and `pools.paused` counts match DB state

### Queue: Usage Logger

- [ ] SessionAgent sends usage message to `USAGE_LOG_QUEUE` after LLM response
- [ ] Queue handler validates message schema, drops invalid messages
- [ ] Valid messages batch-inserted into `usage_log` table
- [ ] DB failure causes `batch.retryAll()` (messages retry)
- [ ] `usage_log.request_index` matches message value

### Queue: Credit Processor

- [ ] SessionAgent sends credit settlement message on session end
- [ ] Actual cost calculated correctly: `(input/1000)*rateIn + (output/1000)*rateOut`
- [ ] Consumer `credit_balance.reserved` decremented by `creditsReserved`
- [ ] Consumer `credit_balance.available` decremented by actual cost
- [ ] Provider `credit_balance.available` incremented by actual cost
- [ ] Refund difference added back to consumer if reserved > actual
- [ ] `credit_ledger` entries created for both consumer and provider
- [ ] `llm_session.credits_charged` updated to actual cost
- [ ] Invalid messages acked (not retried)
- [ ] Settlement failure retries individual message

### Frontend: Marketplace

- [ ] `/pools` renders pool cards with name, reputation, price, models
- [ ] Filter by use case updates URL search params and re-fetches
- [ ] Filter by model filters results
- [ ] Sort dropdown changes ordering
- [ ] Pagination prev/next buttons work, disabled at boundaries
- [ ] Pool card links to `/pools/:poolId` detail page
- [ ] SSR: page loads with data (no loading flash)

### Frontend: Pool Detail

- [ ] `/pools/:poolId` shows full pool info
- [ ] Provider reputation displayed
- [ ] Availability schedule (days + time) rendered
- [ ] "Start Session" button links to session creation
- [ ] 404 pool shows appropriate error

### Frontend: Provider Dashboard

- [ ] Stats cards show sessions, tokens, earnings, reputation
- [ ] Active sessions listed with consumer and token usage
- [ ] Pool list with pause/resume toggle
- [ ] Pause/resume mutation invalidates pool query cache
- [ ] Earnings table shows credit_ledger entries

### Frontend: Consumer Dashboard

- [ ] Credit balance card shows available and reserved
- [ ] Session history table shows completed sessions with cost
- [ ] Credit history table shows all ledger entries with type badges
- [ ] Both tables paginate correctly
- [ ] SSR: dashboard loads with data pre-fetched

---

## 12. Resolved Questions

- ~~Marketplace auth: require login to browse, or fully public?~~ **RESOLVED: require login. Simplifies queries (user context for credits) and aligns with credit system.**
- ~~Pool availability: filter out pools outside current time window in API, or show with "unavailable" badge?~~ **RESOLVED: show with "unavailable" badge. Better UX — pools remain discoverable.**
- ~~Provider stats: cache in KV (stale OK) or always fresh from DB?~~ **RESOLVED: always fresh from DB for MVP. No KV caching complexity.**
- ~~Credit settlement: what if DB transaction fails after 3 retries? DLQ alerting strategy?~~ **RESOLVED: `console.error` with structured log — triggers CF Workers alert. No separate DLQ for MVP.**
- ~~Dashboard refresh: manual refetch button, or polling interval?~~ **RESOLVED: manual refetch button for MVP. No polling complexity.**
- ~~Pool card: show provider username/display name or keep anonymous?~~ **RESOLVED: show provider display name. Builds trust.**
- ~~`maxConcurrentSessions` enforcement: checked in marketplace query (exclude full pools) or only on session create?~~ **RESOLVED: check on session create only. Simpler, real-time accurate.**
- ~~KV prefix purge at scale: switch to versioned keys if >1000 cached variants?~~ **RESOLVED: not needed at MVP scale. Revisit if >1000 cached variants.**

