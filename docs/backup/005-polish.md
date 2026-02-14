# Phase 5 -- Polish

## Overview

Hardening pass across the session lifecycle. Adds token budget warnings, session summarize/export, time-window enforcement via DO alarms, rate limiting, API key encryption, provider reputation, idle timeout, and error handling improvements.

All features are independently implementable and testable. Each section specifies the exact file locations, types, and logic required.

## Context & Background

Phases 1-4 deliver the working session loop: schema, agents, marketplace, CLI proxy. Phase 5 makes it production-ready -- users get warned before hitting limits, sessions auto-end on schedule, API keys are encrypted at rest, and the system handles failure gracefully.

## Goals & Non-Goals

**Goals**
- Warn consumers at 80%/95% token budget
- Summarize/compress/export sessions before data loss
- Enforce pool time windows and idle timeouts via DO alarms
- Rate limit API and WS message throughput
- Encrypt provider API keys at rest (AES-256-GCM)
- Track provider reputation on disconnect
- Graceful handling of Claude API and WS failures

**Non-Goals**
- Content moderation
- Platform fee logic
- Payment/billing integration
- Key rotation automation (documented, not implemented)

---

## 1. Token Budget Warnings

### WS Message Types

```
packages/data-ops/src/zod-schema/ws-messages.ts
```

```ts
const SessionWarningSchema = z.object({
  type: z.literal("session_warning"),
  reason: z.enum(["budget", "time_window", "idle"]), // discriminator
  level: z.enum(["warning", "critical"]),
  percentUsed: z.number(), // budget: output % used; time/idle: 0
  tokensRemaining: z.number(), // budget: output remaining; time/idle: 0
  message: z.string(),
})

type SessionWarning = z.infer<typeof SessionWarningSchema>
```

Two distinct messages:
- `level: "warning"` at 80% -- informational, consumer can continue
- `level: "critical"` at 95% -- auto-prompts save/summarize modal

### SessionAgent Implementation

```
apps/data-service/src/agents/session-agent.ts
```

Token check runs after every LLM response, inside `onMessage` after token metering:

```ts
// maxTokensBudget = output tokens only
private checkBudgetThresholds(): void {
  const { outputTokensUsed, maxTokensBudget } = this.state
  const pct = outputTokensUsed / maxTokensBudget

  if (pct >= 0.95 && !this.state.criticalWarningSent) {
    this.setState({ ...this.state, criticalWarningSent: true })
    this.broadcast({
      type: "session_warning",
      reason: "budget",
      level: "critical",
      percentUsed: Math.round(pct * 100),
      tokensRemaining: maxTokensBudget - outputTokensUsed,
      message: "95% budget used. Save your conversation now.",
    })
    return
  }

  if (pct >= 0.80 && !this.state.warningSent) {
    this.setState({ ...this.state, warningSent: true })
    this.broadcast({
      type: "session_warning",
      reason: "budget",
      level: "warning",
      percentUsed: Math.round(pct * 100),
      tokensRemaining: maxTokensBudget - outputTokensUsed,
      message: "80% budget used.",
    })
  }
}
```

State additions to `SessionState`:

```ts
interface SessionState {
  // ... existing fields
  warningSent: boolean
  criticalWarningSent: boolean
}
```

### Edge Case: Rapid Consumption

A single large response could jump from <80% to >95%. The check handles this: the 95% branch runs first (early return), so if both thresholds are crossed simultaneously only the critical warning fires. The `warningSent` flag stays false -- acceptable, since the critical warning supersedes it.

### UI Components

```
apps/user-application/src/components/session/token-warning-banner.tsx
```

```tsx
interface TokenWarningBannerProps {
  level: "warning" | "critical"
  percentUsed: number
  tokensRemaining: number
  onSave: () => void
}

export function TokenWarningBanner({ level, percentUsed, tokensRemaining, onSave }: TokenWarningBannerProps) {
  // level === "warning": amber Alert banner, dismissible
  // level === "critical": red Alert banner, non-dismissible, includes "Save Now" button
}
```

Display logic in session chat route (`session/$sessionId.tsx`):
- Listen for `session_warning` WS messages via `useAgent` state sync
- `warning` level: render amber banner above chat, dismiss on click
- `critical` level: render red sticky banner + auto-open save/export dialog

---

## 2. Session Summarize/Compress

### Endpoint

```
POST /api/sessions/:id/summarize
```

**Handler**: `apps/data-service/src/hono/handlers/session-handlers.ts`
**Service**: `apps/data-service/src/hono/services/session-service.ts`

```ts
// handler
sessions.post(
  '/:id/summarize',
  authMiddleware,
  zValidator('param', IdParamSchema),
  zValidator('json', SummarizeRequestSchema),
  async (c) => {
    const { id } = c.req.valid('param')
    const { format } = c.req.valid('json')
    const result = await sessionService.summarizeSession(c.env, id, format)
    return c.json(result)
  }
)
```

### Request/Response Schemas

```
packages/data-ops/src/zod-schema/session.ts
```

```ts
const SummarizeRequestSchema = z.object({
  format: z.enum(["md", "json"]).default("md"),
})

const SummarizeResponseSchema = z.object({
  sessionId: z.string(),
  summary: z.string(),
  format: z.enum(["md", "json"]),
  originalMessageCount: z.number(),
  compressedMessageCount: z.number(),
  summaryTokensUsed: z.number(),
})
```

### Summarization Strategy

Executed via `@callable()` on SessionAgent (has conversation in state):

```ts
@callable()
async summarizeSession(format: "md" | "json"): Promise<SummarizeResponse> {
  const messages = this.state.history
  const summaryPrompt = buildSummaryPrompt(messages)

  // Always use Haiku for summaries (cheapest model)
  const summary = await this.callClaudeForSummary(summaryPrompt, "claude-haiku-4-5-20251001")

  if (format === "json") {
    return { /* structured JSON export */ }
  }
  return { /* markdown export */ }
}
```

**Summary prompt template**:
```
Summarize this conversation concisely. Preserve key decisions, code snippets, and action items.

<conversation>
{messages}
</conversation>
```

### Compress Strategy

After summarization, compress replaces the full history:

```ts
@callable()
async compressSession(): Promise<void> {
  const summary = await this.summarizeSession("md")
  const recentN = this.state.history.slice(-5) // keep last 5 messages
  this.setState({
    ...this.state,
    history: [
      { role: "system", content: `[Session Summary]\n${summary.summary}` },
      ...recentN,
    ],
    compressed: true,
  })
}
```

### Token Budget for Summarization

Reserve a fixed 2000-token budget for the summarization call itself. This comes from the session's remaining budget. If remaining tokens < 2000, use whatever is left -- the summary will be shorter. The summarization call uses the same metering path so tokens are tracked.

### Export Formats

**Markdown**:
```md
# Session Export - {sessionId}
Date: {timestamp}
Pool: {poolName}
Tokens used: {input}/{output}

## Summary
{summary text}

## Conversation
### User
{message}

### Assistant
{message}
...
```

**JSON**:
```json
{
  "sessionId": "...",
  "exportedAt": "...",
  "pool": { "id": "...", "name": "..." },
  "usage": { "inputTokens": 0, "outputTokens": 0 },
  "summary": "...",
  "messages": [{ "role": "user", "content": "..." }, ...]
}
```

### UI: Save/Export Modal

```
apps/user-application/src/components/session/export-dialog.tsx
```

Triggered by:
1. User clicks "Export" button in session toolbar
2. Auto-triggered at 95% warning
3. Auto-triggered before session end (time window, idle, manual end)

```tsx
interface ExportDialogProps {
  sessionId: string
  open: boolean
  onOpenChange: (open: boolean) => void
}

export function ExportDialog({ sessionId, open, onOpenChange }: ExportDialogProps) {
  // Radio group: "Full Conversation (MD)" | "Full Conversation (JSON)" | "Summary Only (MD)"
  // "Compress & Continue" button (if session still active)
  // "Download" button
}
```

Uses Radix `Dialog.Root` per UI rules. Format selection via radio group. Download triggers a blob download in-browser.

---

## 3. Time-Window Enforcement via DO Alarms

### Pool Schedule Validation on Session Create

```
apps/data-service/src/hono/services/session-service.ts
```

Before creating a session, validate the current time falls within the pool's schedule:

```ts
function isWithinPoolSchedule(pool: TokenPool): boolean {
  const now = new Date()
  const utcDay = now.getUTCDay() // 0=Sun
  const utcTime = `${String(now.getUTCHours()).padStart(2, '0')}:${String(now.getUTCMinutes()).padStart(2, '0')}`

  if (!pool.availableDays.includes(utcDay)) return false
  if (utcTime < pool.availableFrom || utcTime >= pool.availableTo) return false
  return true
}
```

Reject with 409:
```json
{ "error": "Pool not available at current time", "code": "OUTSIDE_TIME_WINDOW" }
```

### SessionAgent Alarm Setup

On session creation, SessionAgent schedules an alarm at the pool's `available_to` time:

```ts
async onStart() {
  const closingTime = this.getNextClosingTime(
    this.state.poolAvailableTo,
    this.state.poolAvailableDays
  )

  // 5-min warning before close
  this.schedule(
    new Date(closingTime.getTime() - 5 * 60 * 1000),
    "warnTimeWindowClosing"
  )

  // Hard close at window end
  this.schedule(closingTime, "endByTimeWindow")
}

async warnTimeWindowClosing() {
  this.broadcast({
    type: "session_warning",
    reason: "time_window",
    level: "critical",
    percentUsed: 0,
    tokensRemaining: 0,
    message: "Pool closing in 5 minutes. Save your work.",
  })
}

async endByTimeWindow() {
  // Auto-trigger export dialog on client (via WS message)
  this.broadcast({ type: "session_ending", reason: "time_window", gracePeriodMs: 30000 })

  // 30s grace period for user to save
  this.schedule(new Date(Date.now() + 30_000), "forceEndSession")
}

async forceEndSession() {
  await this.endSession("time_window")
}
```

### Multi-Schedule Setup

Each SessionAgent manages up to 4 concurrent schedules:
1. **Time window warning** -- 5min before `available_to`
2. **Time window close** -- at `available_to`
3. **Force end** -- 30s after close (grace period)
4. **Idle timeout** -- 30min rolling (see section 7)

All use agents-sdk `this.schedule()`. Schedules persist through hibernation.

### Timezone Handling

All times stored and compared in UTC (per PLAN.md). `available_from`/`available_to` are UTC time strings ("09:00"/"17:00"). `available_days` are ISO day numbers (0=Sun). No timezone conversion needed.

### `getNextClosingTime` Helper

```ts
private getNextClosingTime(availableTo: string, availableDays: number[]): Date {
  const now = new Date()
  const [hours, minutes] = availableTo.split(':').map(Number)
  const closing = new Date(now)
  closing.setUTCHours(hours, minutes, 0, 0)

  // If closing time already passed today, next available day
  if (closing <= now) {
    closing.setUTCDate(closing.getUTCDate() + 1)
    while (!availableDays.includes(closing.getUTCDay())) {
      closing.setUTCDate(closing.getUTCDate() + 1)
    }
  }

  return closing
}
```

---

## 4. Rate Limiting

### Tiers

| Scope | Limit | Window | Key |
|-------|-------|--------|-----|
| Per-user API | 60 req | 1 min | `user:{userId}` |
| Per-session WS messages | 30 msg | 1 min | `session:{sessionId}` |
| Per-provider concurrent sessions | 5 max | -- | `provider:{providerId}` |

### Implementation: In-Memory (MVP), KV Later

MVP uses in-memory `Map` -- simple, no extra bindings. Sufficient for single-Worker deploys. Migrate to KV-backed counters if multi-instance becomes an issue.

Extend the existing `rate-limiter.ts` pattern. Currently uses in-memory `Map` keyed by IP. Phase 5 adds user-aware rate limiting.

**API rate limiter** (per-user, in Hono middleware):

```
apps/data-service/src/hono/middleware/rate-limiter.ts
```

```ts
interface RateLimitConfig {
  windowMs: number
  maxRequests: number
  keyFn: (c: Context) => string
}

export const rateLimiter = (config: RateLimitConfig): MiddlewareHandler => {
  return async (c, next) => {
    const key = config.keyFn(c)
    const now = Date.now()

    // ... same Map-based logic, keyed by `key` instead of IP

    if (record.count >= config.maxRequests) {
      const retryAfter = Math.ceil((record.resetTime - now) / 1000)
      c.header('Retry-After', String(retryAfter))
      c.header('X-RateLimit-Limit', String(config.maxRequests))
      c.header('X-RateLimit-Remaining', '0')
      c.header('X-RateLimit-Reset', String(Math.ceil(record.resetTime / 1000)))
      return c.json({
        error: 'Too many requests',
        code: 'RATE_LIMITED',
        retryAfter,
      }, 429)
    }

    c.header('X-RateLimit-Limit', String(config.maxRequests))
    c.header('X-RateLimit-Remaining', String(config.maxRequests - record.count))
    c.header('X-RateLimit-Reset', String(Math.ceil(record.resetTime / 1000)))

    record.count++
    return next()
  }
}
```

Usage in `app.ts`:

```ts
App.use('/api/*', rateLimiter({
  windowMs: 60_000,
  maxRequests: 60,
  keyFn: (c) => `user:${c.get('userId') || c.req.header('cf-connecting-ip') || 'anon'}`,
}))
```

**WS message rate limiter** (per-session, inside SessionAgent):

```ts
private messageTimestamps: number[] = []
private readonly WS_RATE_LIMIT = 30
private readonly WS_RATE_WINDOW_MS = 60_000

private isRateLimited(): boolean {
  const now = Date.now()
  this.messageTimestamps = this.messageTimestamps.filter(t => now - t < this.WS_RATE_WINDOW_MS)
  if (this.messageTimestamps.length >= this.WS_RATE_LIMIT) return true
  this.messageTimestamps.push(now)
  return false
}

onMessage(connection: Connection, message: string) {
  if (this.isRateLimited()) {
    connection.send(JSON.stringify({
      type: "error",
      code: "RATE_LIMITED",
      message: "Too many messages. Wait before sending more.",
    }))
    return
  }
  // ... normal message handling
}
```

**Concurrent session limiter** (per-provider, in session creation service):

```ts
// Uses pg_advisory_xact_lock to prevent TOCTOU race on concurrent session creation.
// Lock key = hash of providerId. Lock released automatically when tx commits/rolls back.
async function checkAndReserveConcurrencySlot(
  providerId: string,
  maxConcurrent: number
): Promise<boolean> {
  const db = getDb()
  return db.transaction(async (tx) => {
    // Advisory lock scoped to this provider (prevents concurrent checks)
    const lockKey = hashStringToInt(providerId)
    await tx.execute(sql`SELECT pg_advisory_xact_lock(${lockKey})`)

    const [{ count: activeCount }] = await tx
      .select({ count: count() })
      .from(llmSession)
      .where(and(
        eq(llmSession.providerId, providerId),
        inArray(llmSession.status, ['pending', 'active'])
      ))

    return activeCount < maxConcurrent
  })
}

function hashStringToInt(s: string): number {
  let hash = 0
  for (let i = 0; i < s.length; i++) {
    hash = ((hash << 5) - hash + s.charCodeAt(i)) | 0
  }
  return Math.abs(hash)
}
```

### 429 Response Shape

```ts
const RateLimitErrorSchema = z.object({
  error: z.literal("Too many requests"),
  code: z.literal("RATE_LIMITED"),
  retryAfter: z.number(),
})
```

---

## 5. API Key Encryption Hardening

> **Prerequisite**: `api_key` mode is blocked in Phases 1-4. This section enables it. After implementing encryption, remove the `api_key` mode guard from provider config service.

### Encryption Utilities

```
apps/data-service/src/hono/utils/encryption.ts
```

```ts
const ALGORITHM = "AES-GCM"
const IV_LENGTH = 12
const TAG_LENGTH = 128

interface EncryptedPayload {
  iv: string    // base64
  data: string  // base64
}

export async function encryptApiKey(plaintext: string, encryptionKey: string): Promise<string> {
  const key = await importKey(encryptionKey)
  const iv = crypto.getRandomValues(new Uint8Array(IV_LENGTH))
  const encoded = new TextEncoder().encode(plaintext)

  const ciphertext = await crypto.subtle.encrypt(
    { name: ALGORITHM, iv, tagLength: TAG_LENGTH },
    key,
    encoded,
  )

  const payload: EncryptedPayload = {
    iv: btoa(String.fromCharCode(...iv)),
    data: btoa(String.fromCharCode(...new Uint8Array(ciphertext))),
  }

  return JSON.stringify(payload)
}

export async function decryptApiKey(encrypted: string, encryptionKey: string): Promise<string> {
  const key = await importKey(encryptionKey)
  const payload: EncryptedPayload = JSON.parse(encrypted)

  const iv = Uint8Array.from(atob(payload.iv), c => c.charCodeAt(0))
  const data = Uint8Array.from(atob(payload.data), c => c.charCodeAt(0))

  const decrypted = await crypto.subtle.decrypt(
    { name: ALGORITHM, iv, tagLength: TAG_LENGTH },
    key,
    data,
  )

  return new TextDecoder().decode(decrypted)
}

async function importKey(raw: string): Promise<CryptoKey> {
  const keyData = Uint8Array.from(atob(raw), c => c.charCodeAt(0))
  return crypto.subtle.importKey("raw", keyData, ALGORITHM, false, ["encrypt", "decrypt"])
}
```

### Worker Secret

```bash
# Generate 256-bit key
openssl rand -base64 32

# Set as secret
wrangler secret put ENCRYPTION_KEY --env staging
wrangler secret put ENCRYPTION_KEY --env production
```

Add to `Env` interface (via `cf-typegen`):
```ts
ENCRYPTION_KEY: string
```

### Integration Points

**Encrypt on write** -- in provider config service when saving API key:

```ts
// services/provider-service.ts
async function saveProviderConfig(env: Env, config: ProviderConfigInput) {
  if (config.apiKey) {
    config.encryptedApiKey = await encryptApiKey(config.apiKey, env.ENCRYPTION_KEY)
    delete config.apiKey // never store plaintext
  }
  // ... insert/update DB
}
```

**Decrypt on read** -- in LLM proxy service when making Claude API call:

```ts
// services/llm-proxy.ts
async function getDecryptedKey(env: Env, providerId: string): Promise<string> {
  const config = await getProviderConfig(providerId)
  if (!config.encryptedApiKey) throw new ApiError("No API key configured", 400)
  return decryptApiKey(config.encryptedApiKey, env.ENCRYPTION_KEY)
}
```

### Security Rules

- `ENCRYPTION_KEY` MUST be a Worker secret, never in `wrangler.jsonc` or `.dev.vars` committed to git
- Decrypted keys held only in local variables, never in state/logs
- `console.log` must never include key material -- add lint rule or code review check
- On key save: encrypt -> store -> verify by decrypting -> confirm

### Key Rotation Strategy (Document Only, Not Implemented)

1. Set `ENCRYPTION_KEY_V2` as new secret
2. Batch job reads all `provider_config` rows
3. Decrypt with `ENCRYPTION_KEY` (v1), re-encrypt with `ENCRYPTION_KEY_V2`
4. Update row with new ciphertext + version marker column
5. After all rows migrated, remove `ENCRYPTION_KEY` secret
6. Rename `ENCRYPTION_KEY_V2` -> `ENCRYPTION_KEY`

Requires adding `encryption_version` column to `provider_config` (default 1).

---

## 6. Provider Reputation System

### Schema

Already defined in PLAN.md on `provider_config`:
```
reputation_score: real, default 5.0, range 0.0-5.0
```

### Reputation Update Logic

```
apps/data-service/src/hono/services/reputation-service.ts
```

```ts
const DISCONNECT_PENALTY = 0.1
const MIN_SCORE = 0.0

export async function applyDisconnectPenalty(providerId: string): Promise<number> {
  const db = getDb()

  const [updated] = await db
    .update(providerConfig)
    .set({
      reputationScore: sql`GREATEST(${MIN_SCORE}, reputation_score - ${DISCONNECT_PENALTY})`,
    })
    .where(eq(providerConfig.id, providerId))
    .returning({ reputationScore: providerConfig.reputationScore })

  return updated.reputationScore
}
```

### Trigger Points

Called from SessionAgent when session ends with disconnect reason:

```ts
async endSession(reason: EndReason) {
  // ... settle credits, persist state

  if (reason === "provider_disconnected") {
    await env.CREDIT_TX_QUEUE.send({
      type: "reputation_penalty",
      providerId: this.state.providerId,
    })
  }
}
```

Processed in credit queue handler (reuse existing queue, add message type):

```ts
// queue-handlers/credit-processor.ts
if (msg.body.type === "reputation_penalty") {
  await applyDisconnectPenalty(msg.body.providerId)
  msg.ack()
}
```

### Marketplace Display

Pool cards in `apps/user-application/src/routes/pools/index.tsx` show reputation:

```tsx
<Badge variant={score >= 4.0 ? "default" : score >= 2.5 ? "secondary" : "destructive"}>
  {score.toFixed(1)} / 5.0
</Badge>
```

---

## 7. Session Idle Timeout

### Parameters

- Idle threshold: 30 minutes
- Warning: 25 minutes (5min before timeout)

### SessionAgent Implementation

```ts
private resetIdleTimer() {
  // Cancel existing idle schedules
  const schedules = this.getSchedules()
  for (const s of schedules) {
    if (s.callback === "warnIdle" || s.callback === "endByIdle") {
      this.cancelSchedule(s.id)
    }
  }

  // Schedule new
  this.schedule(new Date(Date.now() + 25 * 60 * 1000), "warnIdle")
  this.schedule(new Date(Date.now() + 30 * 60 * 1000), "endByIdle")
}

async warnIdle() {
  this.broadcast({
    type: "session_warning",
    reason: "idle",
    level: "warning",
    percentUsed: 0,
    tokensRemaining: 0,
    message: "Session idle for 25 minutes. Will auto-end in 5 minutes.",
  })
}

async endByIdle() {
  this.broadcast({ type: "session_ending", reason: "idle_timeout", gracePeriodMs: 0 })
  await this.endSession("timeout")
}
```

Call `resetIdleTimer()` inside `onMessage` on every incoming user message.

### UI

Reuse `TokenWarningBanner` with idle-specific messaging. On `session_ending` with `reason: "idle_timeout"`, auto-trigger export dialog.

---

## 8. Error Handling Hardening

### Claude API Error Types

```
apps/data-service/src/hono/utils/llm-errors.ts
```

```ts
class LlmApiError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly errorType: LlmErrorType,
    public readonly retryable: boolean,
    public readonly retryAfterMs?: number,
  ) {
    super(message)
    this.name = "LlmApiError"
  }
}

type LlmErrorType =
  | "rate_limited"      // 429
  | "overloaded"        // 529
  | "invalid_key"       // 401
  | "invalid_request"   // 400
  | "server_error"      // 500
  | "timeout"           // request timeout
  | "unknown"
```

### Claude API Error Mapping

```ts
function mapClaudeError(status: number, body: unknown): LlmApiError {
  switch (status) {
    case 429:
      return new LlmApiError(
        "Claude API rate limited",
        429,
        "rate_limited",
        true,
        parseRetryAfter(body),
      )
    case 529:
      return new LlmApiError("Claude API overloaded", 529, "overloaded", true, 30_000)
    case 401:
      return new LlmApiError("Invalid API key", 401, "invalid_key", false)
    case 400:
      return new LlmApiError("Invalid request to Claude", 400, "invalid_request", false)
    default:
      return new LlmApiError("Claude API error", status, status >= 500 ? "server_error" : "unknown", status >= 500)
  }
}
```

### Retry Strategy in LLM Proxy

```ts
// services/llm-proxy.ts
const MAX_RETRIES = 2
const BASE_DELAY_MS = 1000

async function callClaudeWithRetry(request: LlmRequest, env: Env): Promise<LlmResponse> {
  for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
    try {
      return await callClaude(request, env)
    } catch (err) {
      if (!(err instanceof LlmApiError) || !err.retryable || attempt === MAX_RETRIES) throw err
      const delay = err.retryAfterMs ?? BASE_DELAY_MS * Math.pow(2, attempt)
      await new Promise(r => setTimeout(r, delay))
    }
  }
  throw new LlmApiError("Max retries exceeded", 503, "server_error", false)
}
```

### WS Disconnect Recovery

SessionAgent handles consumer WS disconnect:

```ts
onClose(connection: Connection, code: number, reason: string) {
  // Don't immediately end session -- allow reconnect within 60s
  this.schedule(new Date(Date.now() + 60_000), "checkReconnection")
  this.setState({ ...this.state, consumerDisconnected: true })
}

async checkReconnection() {
  if (this.state.consumerDisconnected && this.getConnections().length === 0) {
    await this.endSession("consumer_disconnected")
  }
}

onConnect(connection: Connection, ctx: ConnectionContext) {
  // Reconnection: restore state
  if (this.state.consumerDisconnected) {
    this.setState({ ...this.state, consumerDisconnected: false })
    // Cancel reconnection check
    const schedules = this.getSchedules()
    for (const s of schedules) {
      if (s.callback === "checkReconnection") this.cancelSchedule(s.id)
    }
  }
}
```

### Partial Credit Settlement on Error

When a session ends due to error (LLM failure, disconnect), settle credits for tokens actually used:

```ts
async endSession(reason: EndReason) {
  const actualCost = calculateCredits(
    this.state.inputTokensUsed,
    this.state.outputTokensUsed,
    this.state.creditsPerInputKToken,
    this.state.creditsPerOutputKToken,
  )

  await env.CREDIT_TX_QUEUE.send({
    type: "settle",
    sessionId: this.state.sessionId,
    consumerId: this.state.consumerId,
    providerId: this.state.providerId,
    actualCredits: actualCost,
    reservedCredits: this.state.creditsReserved,
    reason,
  })
}
```

Refund = `reservedCredits - actualCredits`. Always >= 0.

### User-Facing Error Messages via WS

```ts
const SessionErrorSchema = z.object({
  type: z.literal("session_error"),
  code: z.enum([
    "llm_rate_limited",
    "llm_overloaded",
    "llm_invalid_key",
    "llm_error",
    "provider_offline",
    "session_expired",
    "rate_limited",
  ]),
  message: z.string(),
  retryable: z.boolean(),
  retryAfterMs: z.number().optional(),
})
```

Example WS error message to consumer:
```json
{
  "type": "session_error",
  "code": "llm_rate_limited",
  "message": "The AI provider is temporarily busy. Retrying in 5 seconds...",
  "retryable": true,
  "retryAfterMs": 5000
}
```

---

## 9. Wrangler Config Changes

No new bindings needed for Phase 5. Existing bindings used:
- `SESSION_AGENT` / `PROVIDER_AGENT` DOs -- alarm scheduling
- `CREDIT_TX_QUEUE` -- reputation penalty messages
- `POOL_CACHE` KV -- no changes

New secret:
```bash
wrangler secret put ENCRYPTION_KEY --env dev
wrangler secret put ENCRYPTION_KEY --env staging
wrangler secret put ENCRYPTION_KEY --env production
```

---

## 10. File Manifest

| File | Action | Feature |
|------|--------|---------|
| `packages/data-ops/src/zod-schema/ws-messages.ts` | modify | Warning + error message types |
| `packages/data-ops/src/zod-schema/session.ts` | modify | Summarize request/response schemas |
| `apps/data-service/src/agents/session-agent.ts` | modify | Budget checks, alarms, idle, reconnect |
| `apps/data-service/src/hono/handlers/session-handlers.ts` | modify | Summarize endpoint |
| `apps/data-service/src/hono/services/session-service.ts` | modify | Schedule validation, summarize |
| `apps/data-service/src/hono/services/reputation-service.ts` | new | Disconnect penalty |
| `apps/data-service/src/hono/services/llm-proxy.ts` | modify | Retry logic, error mapping |
| `apps/data-service/src/hono/middleware/rate-limiter.ts` | modify | User-aware, headers, key fn |
| `apps/data-service/src/hono/utils/encryption.ts` | new | AES-256-GCM encrypt/decrypt |
| `apps/data-service/src/hono/utils/llm-errors.ts` | new | LLM error types |
| `apps/data-service/src/queues/credit-processor.ts` | modify | Reputation penalty handler |
| `apps/user-application/src/components/session/token-warning-banner.tsx` | new | Warning UI |
| `apps/user-application/src/components/session/export-dialog.tsx` | new | Save/export modal |

---

## 11. Verification Checklist

- [ ] At 80% token budget, consumer receives `session_warning` WS message with `level: "warning"`
- [ ] At 95% token budget, consumer receives `session_warning` WS message with `level: "critical"` + save prompt
- [ ] Single response crossing both 80% and 95% sends only critical warning
- [ ] `POST /api/sessions/:id/summarize` returns markdown summary of conversation
- [ ] `POST /api/sessions/:id/summarize` with `format: "json"` returns structured JSON export
- [ ] Compress replaces history with summary + last 5 messages
- [ ] Session auto-ends when pool time window closes (after 30s grace)
- [ ] 5-min warning sent before pool time window closes
- [ ] `POST /api/sessions` returns 409 when pool is outside available hours
- [ ] `POST /api/sessions` returns 409 when pool is outside available days
- [ ] API returns 429 with `Retry-After` header when user exceeds 60 req/min
- [ ] WS sends error message when consumer exceeds 30 msg/min per session
- [ ] Cannot create session when provider has 5 active sessions (concurrent limit)
- [ ] Rate limit response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` headers
- [ ] Encrypted API key in DB is not readable without `ENCRYPTION_KEY`
- [ ] Decrypting with correct key returns original plaintext
- [ ] Decrypted keys never appear in logs or WS messages
- [ ] Provider reputation decreases by 0.1 on mid-session disconnect
- [ ] Provider reputation never drops below 0.0
- [ ] Reputation score displayed on marketplace pool cards
- [ ] Session auto-ends after 30min idle with no user messages
- [ ] Idle warning sent at 25min
- [ ] Idle timer resets on each user message
- [ ] Claude 429 error results in automatic retry with backoff
- [ ] Claude 529 error results in automatic retry with 30s delay
- [ ] Claude 401 error sends `llm_invalid_key` to consumer, no retry
- [ ] Consumer WS disconnect allows 60s reconnection window
- [ ] Partial credit settlement on error: consumer refunded for unused tokens
- [ ] Export dialog auto-opens on session end (any reason)

---

## 12. Resolved Questions

- ~~Summarization model: same model as session or always use cheapest (Haiku)?~~ **RESOLVED: Haiku. Cheapest model for summaries — cost doesn't justify session model.**
- ~~Compress: keep last 5 messages -- configurable per pool or hardcoded?~~ **RESOLVED: hardcoded MVP.**
- ~~WS reconnection: should conversation history replay on reconnect?~~ **RESOLVED: no — state auto-syncs via `agent.state`.**
- ~~Rate limit per-user: identify by Better Auth session or by IP for unauthenticated routes?~~ **RESOLVED: session if authed, IP fallback.**
- ~~`ENCRYPTION_KEY` for local dev: generate per-developer or share via `.dev.vars`?~~ **RESOLVED: per-developer in `.dev.vars`, gitignored.**
- ~~Idle timeout: 30min fixed or configurable per pool?~~ **RESOLVED: fixed 30min. No per-pool config for MVP.**
- ~~Reputation recovery: any mechanism to restore score, or only ever decreases?~~ **RESOLVED: no recovery. Only penalties. Simple and punitive — incentivizes uptime.**
- ~~Grace period after time window (30s): enough for export of large conversations?~~ **RESOLVED: yes for MVP — export is client-side download of already-in-memory state.**
