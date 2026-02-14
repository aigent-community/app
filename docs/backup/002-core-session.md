# 002 - Core Session (Phase 2)

## Overview

Real-time LLM chat sessions between consumers and providers. SessionAgent (Agents SDK DO) manages per-session state, streams LLM responses, meters tokens, enforces budgets. ProviderAgent bridges CLI agents for local-proxy mode. Credit engine handles reserve/settle economics. Chat UI uses `useAgent()` with live state sync.

Phase 2 delivers API-key mode only. Local-proxy (ProviderAgent WS bridge to CLI) ships in Phase 4.

## Context & Background

Phase 1 delivered schema + auth + provider config + pool CRUD + credit balance init. Phase 2 is the critical path: the first time a consumer can actually use tokens. Everything hinges on the SessionAgent lifecycle and credit atomicity.

## Goals

- Consumer creates session, reserves credits, connects WS, chats with LLM
- Token counting from Claude API `usage` field, live counter in UI
- Budget enforcement at 80%/95% thresholds + hard cap
- Credit settlement on session end (refund excess, pay provider)
- Ephemeral conversation with clear UX warnings + export options
- ProviderAgent scaffolding (online status, heartbeat) for Phase 4

## Non-Goals

- Local-proxy mode (Phase 4)
- Marketplace browse UI (Phase 3)
- Queue-based async credit processing (Phase 3 — Phase 2 settles synchronously)
- Content moderation
- Multi-model switching mid-session

---

## 1. SessionAgent

File: `apps/data-service/src/agents/session-agent.ts`

### 1.1 State Shape

```ts
interface SessionState {
  sessionId: string
  poolId: string
  consumerId: string
  providerId: string
  model: string
  systemPrompt: string | null
  status: "pending" | "active" | "ending" | "completed" | "aborted" | "timeout"
  inputTokensUsed: number
  outputTokensUsed: number
  maxTokensBudget: number
  creditsReserved: number
  creditsCharged: number
  history: ChatMessage[]
  warningsSent: WarningLevel[]
  createdAt: string
  endReason: EndReason | null
}

type WarningLevel = "80" | "95"

type EndReason =
  | "tokens_exhausted"
  | "time_window"
  | "user_ended"
  | "provider_revoked"
  | "provider_disconnected"
  | "error"

interface ChatMessage {
  role: "user" | "assistant"
  content: string
  inputTokens: number
  outputTokens: number
  timestamp: string
}
```

State auto-syncs to connected WS clients via Agents SDK `this.setState()`. Consumer UI reads `agent.state` for live token counters.

### 1.2 Class Structure

```ts
import { Agent, callable } from "agents"
import type { Connection, ConnectionContext } from "agents"
import { streamLlmResponse } from "../services/llm-proxy"
import { extractUsage } from "../services/token-meter"
import { reserveCredits, settleCredits } from "../services/credit-engine"

export class SessionAgent extends Agent<Env, SessionState> {
  initialState: SessionState = {
    sessionId: "",
    poolId: "",
    consumerId: "",
    providerId: "",
    model: "",
    systemPrompt: null,
    status: "pending",
    inputTokensUsed: 0,
    outputTokensUsed: 0,
    maxTokensBudget: 0,
    creditsReserved: 0,
    creditsCharged: 0,
    history: [],
    warningsSent: [],
    createdAt: new Date().toISOString(),
    endReason: null,
  }

  async onStart() {
    // Restore schedules after hibernation wake
    if (this.state.status === "active") {
      this.scheduleEvery(60, "syncCredits")
      this.schedule(30 * 60, "checkIdle")
    }
  }

  onConnect(connection: Connection, ctx: ConnectionContext) {
    // Auth validated at WS upgrade route before reaching DO
    // Transition pending -> active on first connect
    if (this.state.status === "pending") {
      this.setState({ ...this.state, status: "active" })
      this.scheduleEvery(60, "syncCredits")
      this.schedule(30 * 60, "checkIdle")
    }
  }

  onMessage(connection: Connection, message: string | ArrayBuffer) {
    const parsed = ConsumerMessageSchema.safeParse(
      JSON.parse(message as string)
    )
    if (!parsed.success) {
      connection.send(JSON.stringify({
        type: "error",
        message: "Invalid message format",
      }))
      return
    }
    const msg = parsed.data
    switch (msg.type) {
      case "chat_message":
        this.handleChatMessage(connection, msg.content)
        break
      case "end_session":
        this.endSession()
        break
      case "export_session":
        this.handleExport(connection, msg.format)
        break
    }
  }

  onClose(connection: Connection, code: number, reason: string) {
    // Consumer disconnected — don't end session immediately
    // checkIdle schedule handles cleanup after 30m
  }
}
```

### 1.3 @callable Methods

```ts
@callable()
async streamResponse(userMessage: string): Promise<void> {
  if (this.state.status !== "active") {
    throw new Error("Session not active")
  }

  // maxTokensBudget = output tokens only
  const outputUsed = this.state.outputTokensUsed
  if (outputUsed >= this.state.maxTokensBudget) {
    await this.endSessionInternal("tokens_exhausted")
    return
  }

  // Add user message to history
  const userEntry: ChatMessage = {
    role: "user",
    content: userMessage,
    inputTokens: 0,
    outputTokens: 0,
    timestamp: new Date().toISOString(),
  }

  this.setState({
    ...this.state,
    history: [...this.state.history, userEntry],
  })

  // Sliding window: send last 50 messages to Claude (full history kept in state for export)
  const MAX_CONTEXT_MESSAGES = 50
  const contextMessages = this.state.history.slice(-MAX_CONTEXT_MESSAGES).map(m => ({
    role: m.role,
    content: m.content,
  }))

  const result = await streamLlmResponse({
    model: this.state.model,
    messages: contextMessages,
    system: this.state.systemPrompt ?? undefined,
    apiKey: this.env.ANTHROPIC_API_KEY,
    maxTokens: Math.min(
      4096,
      this.state.maxTokensBudget - outputUsed
    ),
    onChunk: (chunk: string) => {
      this.broadcast(JSON.stringify({
        type: "stream_chunk",
        content: chunk,
      } satisfies StreamChunkMessage))
    },
  })

  // Extract usage and update state
  const usage = extractUsage(result)
  const assistantEntry: ChatMessage = {
    role: "assistant",
    content: result.fullText,
    inputTokens: usage.inputTokens,
    outputTokens: usage.outputTokens,
    timestamp: new Date().toISOString(),
  }

  const newInputTotal = this.state.inputTokensUsed + usage.inputTokens
  const newOutputTotal = this.state.outputTokensUsed + usage.outputTokens

  this.setState({
    ...this.state,
    inputTokensUsed: newInputTotal,
    outputTokensUsed: newOutputTotal,
    history: [...this.state.history, assistantEntry],
  })

  // Broadcast token update
  this.broadcast(JSON.stringify({
    type: "token_update",
    inputTokensUsed: newInputTotal,
    outputTokensUsed: newOutputTotal,
    maxTokensBudget: this.state.maxTokensBudget,
  } satisfies TokenUpdateMessage))

  // Log usage to SQLite
  this.sql.exec(
    `INSERT INTO usage_log (request_index, input_tokens, output_tokens, model, latency_ms, created_at)
     VALUES (?, ?, ?, ?, ?, ?)`,
    this.state.history.filter(m => m.role === "assistant").length,
    usage.inputTokens,
    usage.outputTokens,
    this.state.model,
    usage.latencyMs,
    new Date().toISOString()
  )

  // Budget warnings (output tokens only)
  this.checkBudgetWarnings(newOutputTotal)
}

@callable()
async endSession(): Promise<void> {
  await this.endSessionInternal("user_ended")
}

@callable()
async summarizeSession(): Promise<string> {
  if (this.state.history.length === 0) return "No conversation to summarize."

  // Cap summary output to remaining budget (or 1024, whichever is smaller)
  const remaining = this.state.maxTokensBudget - this.state.outputTokensUsed
  if (remaining <= 0) return "Budget exhausted — cannot generate summary."
  const summaryMaxTokens = Math.min(1024, remaining)

  const result = await streamLlmResponse({
    model: this.state.model,
    messages: [
      ...this.state.history.map(m => ({ role: m.role, content: m.content })),
      {
        role: "user",
        content: "Summarize this conversation concisely, capturing key points and decisions.",
      },
    ],
    apiKey: this.env.ANTHROPIC_API_KEY,
    maxTokens: summaryMaxTokens,
    onChunk: () => {}, // no streaming for summary
  })

  const usage = extractUsage(result)
  const newOutputTotal = this.state.outputTokensUsed + usage.outputTokens
  this.setState({
    ...this.state,
    inputTokensUsed: this.state.inputTokensUsed + usage.inputTokens,
    outputTokensUsed: newOutputTotal,
  })

  // Check if summary pushed us past thresholds
  this.checkBudgetWarnings(newOutputTotal)

  return result.fullText
}

@callable()
async exportSession(format: "md" | "json"): Promise<string> {
  if (format === "json") {
    return JSON.stringify({
      sessionId: this.state.sessionId,
      model: this.state.model,
      messages: this.state.history,
      tokenUsage: {
        input: this.state.inputTokensUsed,
        output: this.state.outputTokensUsed,
      },
      exportedAt: new Date().toISOString(),
    }, null, 2)
  }

  // Markdown
  const lines = [
    `# Session ${this.state.sessionId}`,
    `Model: ${this.state.model}`,
    `Tokens: ${this.state.inputTokensUsed} in / ${this.state.outputTokensUsed} out`,
    "",
  ]
  for (const msg of this.state.history) {
    lines.push(`## ${msg.role === "user" ? "User" : "Assistant"}`)
    lines.push(msg.content)
    lines.push("")
  }
  return lines.join("\n")
}
```

### 1.4 Internal Methods

```ts
// totalOutputUsed = outputTokensUsed only (maxTokensBudget is output-only)
private checkBudgetWarnings(totalOutputUsed: number) {
  const pct = totalOutputUsed / this.state.maxTokensBudget
  if (pct >= 0.95 && !this.state.warningsSent.includes("95")) {
    this.setState({
      ...this.state,
      warningsSent: [...this.state.warningsSent, "95"],
    })
    this.broadcast(JSON.stringify({
      type: "session_warning",
      level: "95",
      message: "95% of token budget used. Save/export your conversation now.",
      tokensRemaining: this.state.maxTokensBudget - totalUsed,
    } satisfies SessionWarningMessage))
  } else if (pct >= 0.80 && !this.state.warningsSent.includes("80")) {
    this.setState({
      ...this.state,
      warningsSent: [...this.state.warningsSent, "80"],
    })
    this.broadcast(JSON.stringify({
      type: "session_warning",
      level: "80",
      message: "80% of token budget used.",
      tokensRemaining: this.state.maxTokensBudget - totalUsed,
    } satisfies SessionWarningMessage))
  }
}

private async endSessionInternal(reason: EndReason) {
  if (this.state.status === "completed" || this.state.status === "ending") {
    return
  }

  this.setState({ ...this.state, status: "ending" })

  // Settle credits
  const settlement = await settleCredits({
    sessionId: this.state.sessionId,
    consumerId: this.state.consumerId,
    providerId: this.state.providerId,
    poolId: this.state.poolId,
    inputTokensUsed: this.state.inputTokensUsed,
    outputTokensUsed: this.state.outputTokensUsed,
    creditsReserved: this.state.creditsReserved,
  })

  this.setState({
    ...this.state,
    status: "completed",
    creditsCharged: settlement.creditsCharged,
    endReason: reason,
  })

  // Persist final session state to Postgres
  await this.persistSessionToDb()

  // Broadcast session ended
  this.broadcast(JSON.stringify({
    type: "session_ended",
    reason,
    creditsCharged: settlement.creditsCharged,
    creditsRefunded: settlement.creditsRefunded,
  } satisfies SessionEndedMessage))

  // Cancel all schedules
  const schedules = await this.getSchedules()
  for (const s of schedules) {
    await this.cancelSchedule(s.id)
  }
}

private async persistSessionToDb() {
  // Update llm_session row with final state
  await updateSession(this.state.sessionId, {
    status: this.state.status,
    inputTokensUsed: this.state.inputTokensUsed,
    outputTokensUsed: this.state.outputTokensUsed,
    creditsCharged: this.state.creditsCharged,
    endReason: this.state.endReason,
  })
}

private async handleChatMessage(connection: Connection, content: string) {
  try {
    await this.streamResponse(content)
  } catch (error) {
    connection.send(JSON.stringify({
      type: "stream_error",
      error: error instanceof Error ? error.message : "Unknown error",
    }))
  }
}

private async handleExport(connection: Connection, format: "md" | "json") {
  try {
    const data = await this.exportSession(format)
    connection.send(JSON.stringify({
      type: "export_result",
      format,
      data,
    } satisfies ExportResultMessage))
  } catch (error) {
    connection.send(JSON.stringify({
      type: "stream_error",
      error: error instanceof Error ? error.message : "Export failed",
    }))
  }
}
```

### 1.5 Schedule Handlers

```ts
// Called every 60s — sync token usage to Postgres
async syncCredits() {
  if (this.state.status !== "active") return
  await updateSession(this.state.sessionId, {
    inputTokensUsed: this.state.inputTokensUsed,
    outputTokensUsed: this.state.outputTokensUsed,
  })
}

// Called once after 30m — check if session is idle
async checkIdle() {
  if (this.state.status !== "active") return
  const lastMessage = this.state.history[this.state.history.length - 1]
  if (!lastMessage) {
    await this.endSessionInternal("timeout")
    return
  }
  const lastTs = new Date(lastMessage.timestamp).getTime()
  const thirtyMinAgo = Date.now() - 30 * 60 * 1000
  if (lastTs < thirtyMinAgo) {
    await this.endSessionInternal("timeout")
  } else {
    // Reschedule
    const remaining = lastTs + 30 * 60 * 1000 - Date.now()
    this.schedule(Math.ceil(remaining / 1000), "checkIdle")
  }
}
```

### 1.6 SQLite Schema (Agent-local)

Created in `onStart` if not exists:

```ts
async onStart() {
  this.sql.exec(`
    CREATE TABLE IF NOT EXISTS usage_log (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      request_index INTEGER NOT NULL,
      input_tokens INTEGER NOT NULL,
      output_tokens INTEGER NOT NULL,
      model TEXT NOT NULL,
      latency_ms INTEGER NOT NULL,
      created_at TEXT NOT NULL
    )
  `)
  // ... restore schedules
}
```

---

## 2. ProviderAgent

File: `apps/data-service/src/agents/provider-agent.ts`

Phase 2 scaffolding only. Full CLI bridge in Phase 4.

### 2.1 State Shape

```ts
interface ProviderState {
  providerId: string
  userId: string
  online: boolean
  activeRequests: number
  lastHeartbeat: string | null
  stats: ProviderStats
}

interface ProviderStats {
  totalRequests: number
  totalInputTokens: number
  totalOutputTokens: number
  uptime: number
}
```

### 2.2 Class Structure

```ts
import { Agent, callable } from "agents"
import type { Connection, ConnectionContext } from "agents"

export class ProviderAgent extends Agent<Env, ProviderState> {
  initialState: ProviderState = {
    providerId: "",
    userId: "",
    online: false,
    activeRequests: 0,
    lastHeartbeat: null,
    stats: {
      totalRequests: 0,
      totalInputTokens: 0,
      totalOutputTokens: 0,
      uptime: 0,
    },
  }

  async onStart() {
    // If previously online, start heartbeat check
    if (this.state.online) {
      this.scheduleEvery(30, "heartbeat")
    }
  }

  onConnect(connection: Connection, ctx: ConnectionContext) {
    // CLI agent connects
    this.setState({
      ...this.state,
      online: true,
      lastHeartbeat: new Date().toISOString(),
    })
    this.scheduleEvery(30, "heartbeat")
    // Update KV for marketplace queries
    this.env.POOL_CACHE.put(
      `provider:online:${this.state.providerId}`,
      "true",
      { expirationTtl: 120 }
    )
  }

  onMessage(connection: Connection, message: string | ArrayBuffer) {
    const parsed = JSON.parse(message as string)
    if (parsed.type === "heartbeat") {
      this.setState({
        ...this.state,
        lastHeartbeat: new Date().toISOString(),
      })
    }
    // Phase 4: handle LLM response forwarding
  }

  onClose(connection: Connection, code: number, reason: string) {
    this.setState({ ...this.state, online: false })
    this.env.POOL_CACHE.delete(`provider:online:${this.state.providerId}`)
  }

  @callable()
  async isOnline(): Promise<boolean> {
    return this.state.online
  }

  @callable()
  async getStats(): Promise<ProviderStats> {
    return this.state.stats
  }

  // Phase 4: forward LLM request to CLI via WS, stream chunks back to
  // SessionAgent via reverse RPC (getAgentByName). No callback params —
  // @callable() uses JSON-serialized RPC.
  @callable()
  async forwardLlmRequest(request: LlmProxyRequest & { sessionId: string }): Promise<void> {
    throw new Error("Local proxy not implemented — use API key mode")
  }

  // Heartbeat check — if no heartbeat in 90s, mark offline
  async heartbeat() {
    if (!this.state.lastHeartbeat) return
    const last = new Date(this.state.lastHeartbeat).getTime()
    if (Date.now() - last > 90_000) {
      this.setState({ ...this.state, online: false })
      await this.env.POOL_CACHE.delete(
        `provider:online:${this.state.providerId}`
      )
      const schedules = await this.getSchedules()
      for (const s of schedules) {
        await this.cancelSchedule(s.id)
      }
    }
  }
}
```

---

## 3. LLM Proxy Service

File: `apps/data-service/src/hono/services/llm-proxy.ts`

### 3.1 Types

```ts
interface LlmProxyRequest {
  model: string
  messages: Array<{ role: "user" | "assistant"; content: string }>
  system?: string
  apiKey: string
  maxTokens: number
  onChunk: (chunk: string) => void
}

interface LlmProxyResult {
  fullText: string
  usage: {
    inputTokens: number
    outputTokens: number
  }
  model: string
  stopReason: string
  latencyMs: number
}

interface ClaudeStreamEvent {
  type: string
  index?: number
  delta?: { type: string; text?: string }
  message?: {
    usage: { input_tokens: number; output_tokens: number }
    stop_reason: string
    model: string
  }
  usage?: { output_tokens: number }
}
```

### 3.2 Implementation

```ts
const CLAUDE_API_URL = "https://api.anthropic.com/v1/messages"

export async function streamLlmResponse(
  request: LlmProxyRequest
): Promise<LlmProxyResult> {
  const start = Date.now()

  const response = await fetch(CLAUDE_API_URL, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": request.apiKey,
      "anthropic-version": "2023-06-01",
    },
    body: JSON.stringify({
      model: request.model,
      max_tokens: request.maxTokens,
      messages: request.messages,
      ...(request.system && { system: request.system }),
      stream: true,
    }),
  })

  if (!response.ok) {
    const errorBody = await response.text()
    throw new LlmProxyError(
      `Claude API returned ${response.status}`,
      response.status,
      errorBody
    )
  }

  const reader = response.body!.getReader()
  const decoder = new TextDecoder()
  let fullText = ""
  let inputTokens = 0
  let outputTokens = 0
  let stopReason = ""
  let model = request.model
  let buffer = ""

  while (true) {
    const { done, value } = await reader.read()
    if (done) break

    buffer += decoder.decode(value, { stream: true })
    const lines = buffer.split("\n")
    buffer = lines.pop() ?? ""

    for (const line of lines) {
      if (!line.startsWith("data: ")) continue
      const data = line.slice(6).trim()
      if (data === "[DONE]") continue

      const event: ClaudeStreamEvent = JSON.parse(data)

      switch (event.type) {
        case "message_start":
          if (event.message?.usage) {
            inputTokens = event.message.usage.input_tokens
          }
          if (event.message?.model) {
            model = event.message.model
          }
          break

        case "content_block_delta":
          if (event.delta?.text) {
            fullText += event.delta.text
            request.onChunk(event.delta.text)
          }
          break

        case "message_delta":
          if (event.usage) {
            outputTokens = event.usage.output_tokens
          }
          if (event.message?.stop_reason) {
            stopReason = event.message.stop_reason
          }
          // message_delta has stop_reason at top level too
          if ((event as Record<string, unknown>).stop_reason) {
            stopReason = (event as Record<string, unknown>).stop_reason as string
          }
          break
      }
    }
  }

  return {
    fullText,
    usage: { inputTokens, outputTokens },
    model,
    stopReason,
    latencyMs: Date.now() - start,
  }
}
```

### 3.3 Error Class

```ts
export class LlmProxyError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly body: string
  ) {
    super(message)
    this.name = "LlmProxyError"
  }
}
```

---

## 4. Token Meter

File: `apps/data-service/src/hono/services/token-meter.ts`

### 4.1 Types

```ts
interface TokenUsage {
  inputTokens: number
  outputTokens: number
  totalTokens: number
  latencyMs: number
}

interface SessionTokenTotals {
  inputTokensUsed: number
  outputTokensUsed: number
  totalTokensUsed: number
  requestCount: number
}
```

### 4.2 Implementation

```ts
export function extractUsage(result: LlmProxyResult): TokenUsage {
  return {
    inputTokens: result.usage.inputTokens,
    outputTokens: result.usage.outputTokens,
    totalTokens: result.usage.inputTokens + result.usage.outputTokens,
    latencyMs: result.latencyMs,
  }
}

export function accumulateUsage(
  current: SessionTokenTotals,
  usage: TokenUsage
): SessionTokenTotals {
  return {
    inputTokensUsed: current.inputTokensUsed + usage.inputTokens,
    outputTokensUsed: current.outputTokensUsed + usage.outputTokens,
    totalTokensUsed: current.totalTokensUsed + usage.totalTokens,
    requestCount: current.requestCount + 1,
  }
}

export function calculateCreditsUsed(
  inputTokens: number,
  outputTokens: number,
  creditsPerInputKToken: number,
  creditsPerOutputKToken: number
): number {
  return Math.ceil(
    (inputTokens / 1000) * creditsPerInputKToken +
    (outputTokens / 1000) * creditsPerOutputKToken
  )
}

export function getBudgetPercentage(
  totalUsed: number,
  maxBudget: number
): number {
  if (maxBudget === 0) return 100
  return (totalUsed / maxBudget) * 100
}
```

---

## 5. Credit Engine

File: `apps/data-service/src/hono/services/credit-engine.ts`

### 5.1 Types

```ts
interface ReserveCreditsInput {
  consumerId: string
  maxTokensBudget: number
  creditsPerInputKToken: number
  creditsPerOutputKToken: number
}

interface ReserveCreditsResult {
  creditsReserved: number
}

interface SettleCreditsInput {
  sessionId: string
  consumerId: string
  providerId: string
  poolId: string
  inputTokensUsed: number
  outputTokensUsed: number
  creditsReserved: number
}

interface SettleCreditsResult {
  creditsCharged: number
  creditsRefunded: number
  providerEarned: number
}
```

### 5.2 Reserve (called at session creation)

```ts
import { getDb } from "@repo/data-ops/database/setup"
import { creditBalance, creditLedger } from "@repo/data-ops/drizzle/schema"
import { eq, sql } from "drizzle-orm"

export async function reserveCredits(
  input: ReserveCreditsInput
): Promise<ReserveCreditsResult> {
  const db = getDb()

  // Worst-case cost: all budget tokens as output (most expensive rate)
  const maxCredits = calculateCreditsUsed(
    0,
    input.maxTokensBudget,
    input.creditsPerInputKToken,
    input.creditsPerOutputKToken
  )

  // Optimistic lock: atomically check + reserve
  const result = await db
    .update(creditBalance)
    .set({
      available: sql`${creditBalance.available} - ${maxCredits}`,
      reserved: sql`${creditBalance.reserved} + ${maxCredits}`,
    })
    .where(
      sql`${creditBalance.userId} = ${input.consumerId}
          AND ${creditBalance.available} >= ${maxCredits}`
    )
    .returning()

  if (result.length === 0) {
    throw new InsufficientCreditsError(
      "Not enough credits to reserve for this session"
    )
  }

  return { creditsReserved: maxCredits }
}
```

### 5.3 Settle (called on session end)

```ts
export async function settleCredits(
  input: SettleCreditsInput
): Promise<SettleCreditsResult> {
  const db = getDb()

  // Get pool pricing
  const pool = await getPool(input.poolId)
  if (!pool) throw new Error("Pool not found during settlement")

  const actualCredits = calculateCreditsUsed(
    input.inputTokensUsed,
    input.outputTokensUsed,
    pool.creditsPerInputKToken,
    pool.creditsPerOutputKToken
  )

  const refund = input.creditsReserved - actualCredits

  // All mutations in single transaction to prevent credit loss/duplication
  const result = await db.transaction(async (tx) => {
    // Consumer: release reserved, refund excess to available
    await tx
      .update(creditBalance)
      .set({
        reserved: sql`${creditBalance.reserved} - ${input.creditsReserved}`,
        available: sql`${creditBalance.available} + ${refund}`,
      })
      .where(eq(creditBalance.userId, input.consumerId))

    // Consumer ledger entry (balanceAfter computed inside tx)
    const [consumerBal] = await tx
      .select({ available: creditBalance.available })
      .from(creditBalance)
      .where(eq(creditBalance.userId, input.consumerId))

    await tx.insert(creditLedger).values({
      id: generateUlid(),
      userId: input.consumerId,
      amount: -actualCredits,
      type: "consumer_spend",
      sessionId: input.sessionId,
      balanceAfter: consumerBal.available,
    })

    // Provider: add earnings
    await tx
      .update(creditBalance)
      .set({
        available: sql`${creditBalance.available} + ${actualCredits}`,
      })
      .where(eq(creditBalance.userId, input.providerId))

    // Provider ledger entry
    const [providerBal] = await tx
      .select({ available: creditBalance.available })
      .from(creditBalance)
      .where(eq(creditBalance.userId, input.providerId))

    await tx.insert(creditLedger).values({
      id: generateUlid(),
      userId: input.providerId,
      amount: actualCredits,
      type: "provider_earning",
      sessionId: input.sessionId,
      balanceAfter: providerBal.available,
    })

    return { creditsCharged: actualCredits, creditsRefunded: refund, providerEarned: actualCredits }
  })

  return result
}
```

### 5.4 Error Class

```ts
export class InsufficientCreditsError extends Error {
  constructor(message: string) {
    super(message)
    this.name = "InsufficientCreditsError"
  }
}
```

---

## 6. WebSocket Routes

File: `apps/data-service/src/hono/handlers/ws-handlers.ts`

WS upgrades use custom Hono routes (not `routeAgentRequest`) so auth middleware runs *before* the upgrade. This keeps auth consistent with REST endpoints.

### 6.1 Entry Point

File: `apps/data-service/src/index.ts`

```ts
import { WorkerEntrypoint } from "cloudflare:workers"
import { App } from "@/hono/app"
import { initDatabase } from "@repo/data-ops/database/setup"
import { handleScheduled } from "./scheduled"
import { handleQueue } from "./queues"

export { SessionAgent } from "./agents/session-agent"
export { ProviderAgent } from "./agents/provider-agent"

export default class DataService extends WorkerEntrypoint<Env> {
  constructor(ctx: ExecutionContext, env: Env) {
    super(ctx, env)
    initDatabase({
      host: env.DATABASE_HOST,
      username: env.DATABASE_USERNAME,
      password: env.DATABASE_PASSWORD,
    })
  }

  async fetch(request: Request) {
    // All routing (REST + WS) handled by Hono — no routeAgentRequest
    return App.fetch(request, this.env, this.ctx)
  }

  async scheduled(controller: ScheduledController) {
    await handleScheduled(controller, this.env, this.ctx)
  }

  async queue(batch: MessageBatch<ExampleQueueMessage>) {
    await handleQueue(batch, this.env)
  }
}
```

### 6.2 WS Routes (auth before upgrade)

```ts
// apps/data-service/src/hono/handlers/ws-handlers.ts
import { Hono } from "hono"
import { getAgentByName } from "agents"

const ws = new Hono<{ Bindings: Env }>()

// Consumer WS to SessionAgent
ws.get("/session/:sessionId", async (c) => {
  // Validate auth from cookie/header
  const session = await validateWsAuth(c)
  if (!session) {
    return c.json({ error: "Unauthorized" }, 401)
  }

  // Verify consumer owns this session
  const llmSession = await getSessionById(c.req.param("sessionId"))
  if (!llmSession || llmSession.consumerId !== session.user.id) {
    return c.json({ error: "Forbidden" }, 403)
  }

  // Get agent stub and proxy the upgrade
  const agent = await getAgentByName<SessionAgent>(
    c.env.SESSION_AGENT,
    llmSession.id
  )

  // Forward upgrade request to agent
  return agent.fetch(c.req.raw)
})

// Provider WS to ProviderAgent
ws.get("/provider", async (c) => {
  const session = await validateWsAuth(c)
  if (!session) {
    return c.json({ error: "Unauthorized" }, 401)
  }

  const providerConfig = await getProviderConfigByUserId(session.user.id)
  if (!providerConfig) {
    return c.json({ error: "No provider config" }, 404)
  }

  const agent = await getAgentByName<ProviderAgent>(
    c.env.PROVIDER_AGENT,
    providerConfig.id
  )

  return agent.fetch(c.req.raw)
})

export default ws
```

### 6.3 Auth Validation

```ts
import { getAuth } from "@repo/data-ops/auth/server"

async function validateWsAuth(c: Context<{ Bindings: Env }>) {
  // WS upgrade can carry auth via:
  // 1. Cookie (same-origin browser connections)
  // 2. Sec-WebSocket-Protocol header (token subprotocol)
  const auth = getAuth()
  const session = await auth.api.getSession({
    headers: c.req.raw.headers,
  })
  return session
}
```

### 6.4 Register in App

```ts
// apps/data-service/src/hono/app.ts
import ws from "./handlers/ws-handlers"

App.route("/ws", ws)
```

---

## 7. Session API Routes

File: `apps/data-service/src/hono/handlers/session-handlers.ts`

### 7.1 Route Definitions

```ts
import { Hono } from "hono"
import { zValidator } from "@hono/zod-validator"
import {
  SessionCreateRequestSchema,
  SessionIdParamSchema,
} from "@repo/data-ops/zod-schema/session"
import { betterAuthMiddleware } from "../middleware/auth"
import * as sessionService from "../services/session-service"

const sessions = new Hono<{ Bindings: Env }>()

// All session routes require auth
sessions.use("*", betterAuthMiddleware())

// POST /api/sessions — create session + reserve credits
sessions.post(
  "/",
  zValidator("json", SessionCreateRequestSchema),
  async (c) => {
    const data = c.req.valid("json")
    const userId = c.get("userId")
    const result = await sessionService.createSession(userId, data)
    return c.json(result, 201)
  }
)

// GET /api/sessions (list — used by provider + consumer dashboards)
sessions.get(
  "/",
  zValidator("query", SessionListQuerySchema),
  async (c) => {
    const query = c.req.valid("query")
    const userId = c.get("userId")
    const result = await sessionService.listSessions(userId, query)
    return c.json(result)
  }
)

// GET /api/sessions/:id
sessions.get(
  "/:id",
  zValidator("param", SessionIdParamSchema),
  async (c) => {
    const { id } = c.req.valid("param")
    const userId = c.get("userId")
    return c.json(await sessionService.getSession(id, userId))
  }
)

// POST /api/sessions/:id/end
sessions.post(
  "/:id/end",
  zValidator("param", SessionIdParamSchema),
  async (c) => {
    const { id } = c.req.valid("param")
    const userId = c.get("userId")
    await sessionService.endSession(id, userId, c.env)
    return c.json({ ok: true })
  }
)

// POST /api/sessions/:id/summarize
sessions.post(
  "/:id/summarize",
  zValidator("param", SessionIdParamSchema),
  async (c) => {
    const { id } = c.req.valid("param")
    const userId = c.get("userId")
    const summary = await sessionService.summarizeSession(id, userId, c.env)
    return c.json({ summary })
  }
)

export default sessions
```

### 7.2 Service Layer

File: `apps/data-service/src/hono/services/session-service.ts`

```ts
import { HTTPException } from "hono/http-exception"
import { getAgentByName } from "agents"
import { reserveCredits } from "./credit-engine"
import type { SessionCreateInput } from "@repo/data-ops/zod-schema/session"

export async function createSession(
  consumerId: string,
  input: SessionCreateInput
) {
  // Validate pool exists, is active, has capacity
  const pool = await getPool(input.poolId)
  if (!pool || pool.status !== "active") {
    throw new HTTPException(404, { message: "Pool not found or inactive" })
  }

  // Lazy daily token reset: if dailyTokensResetAt is a different UTC day, zero the counter
  const today = new Date().toISOString().slice(0, 10)
  const resetDay = pool.dailyTokensResetAt.toISOString().slice(0, 10)
  if (today !== resetDay) {
    await updatePool(pool.id, { dailyTokensUsed: 0, dailyTokensResetAt: new Date() })
    pool.dailyTokensUsed = 0
  }

  // Check concurrent session limit
  const activeCount = await countActiveSessionsByPool(pool.id)
  if (activeCount >= pool.maxConcurrentSessions) {
    throw new HTTPException(429, { message: "Pool at max concurrent sessions" })
  }

  // Reserve credits
  const reservation = await reserveCredits({
    consumerId,
    maxTokensBudget: input.maxTokensBudget,
    creditsPerInputKToken: pool.creditsPerInputKToken,
    creditsPerOutputKToken: pool.creditsPerOutputKToken,
  })

  // Pick model from pool's allowed list
  const model = input.model ?? pool.allowedModels[0]
  if (!pool.allowedModels.includes(model)) {
    throw new HTTPException(400, { message: "Model not allowed by this pool" })
  }

  // Insert session row
  const session = await insertSession({
    id: generateUlid(),
    poolId: pool.id,
    consumerId,
    providerId: pool.providerId,
    status: "pending",
    model,
    maxTokensBudget: input.maxTokensBudget,
    creditsReserved: reservation.creditsReserved,
  })

  return session
}

export async function listSessions(
  userId: string,
  query: SessionListQuery
) {
  const db = getDb()
  const conditions = []

  // filter by role: consumer sees their consumed sessions, provider sees their provided
  if (query.role === "provider") {
    conditions.push(eq(llmSession.providerId, userId))
  } else {
    conditions.push(eq(llmSession.consumerId, userId))
  }

  if (query.status) {
    conditions.push(eq(llmSession.status, query.status))
  }

  const where = and(...conditions)

  const [data, [{ total }]] = await Promise.all([
    db
      .select()
      .from(llmSession)
      .where(where)
      .orderBy(desc(llmSession.createdAt))
      .limit(query.limit)
      .offset(query.offset),
    db
      .select({ total: count() })
      .from(llmSession)
      .where(where),
  ])

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

export async function endSession(
  sessionId: string,
  userId: string,
  env: Env
) {
  const session = await getSessionById(sessionId)
  if (!session) throw new HTTPException(404, { message: "Session not found" })
  if (session.consumerId !== userId) {
    throw new HTTPException(403, { message: "Not your session" })
  }

  const agent = await getAgentByName<SessionAgent>(
    env.SESSION_AGENT,
    sessionId
  )
  await agent.endSession()
}

export async function summarizeSession(
  sessionId: string,
  userId: string,
  env: Env
): Promise<string> {
  const session = await getSessionById(sessionId)
  if (!session) throw new HTTPException(404, { message: "Session not found" })
  if (session.consumerId !== userId) {
    throw new HTTPException(403, { message: "Not your session" })
  }

  const agent = await getAgentByName<SessionAgent>(
    env.SESSION_AGENT,
    sessionId
  )
  return agent.summarizeSession()
}
```

### 7.3 Register in App

```ts
// apps/data-service/src/hono/app.ts
import sessions from "./handlers/session-handlers"

App.route("/api/sessions", sessions)
```

---

## 8. Frontend Chat UI

File: `apps/user-application/src/routes/_auth/session/$sessionId.tsx`

### 8.1 Route Definition

```tsx
import { createFileRoute } from "@tanstack/react-router"
import { useAgent } from "agents/react"
import { useState, useRef, useEffect } from "react"
import { TokenCounter } from "@/components/session/token-counter"
import { EphemeralBanner } from "@/components/session/ephemeral-banner"
import { ExportModal } from "@/components/session/export-modal"
import { WarningBanner } from "@/components/session/warning-banner"
import { ChatMessages } from "@/components/session/chat-messages"
import { ChatInput } from "@/components/session/chat-input"

export const Route = createFileRoute("/_auth/session/$sessionId")({
  component: SessionPage,
})

function SessionPage() {
  const { sessionId } = Route.useParams()
  const [streamingContent, setStreamingContent] = useState("")
  const [showExportModal, setShowExportModal] = useState(false)
  const [warning, setWarning] = useState<SessionWarningMessage | null>(null)
  const messagesEndRef = useRef<HTMLDivElement>(null)

  const agent = useAgent({
    agent: "SessionAgent",
    name: sessionId,
    onMessage: (message: MessageEvent) => {
      const data = JSON.parse(message.data)
      switch (data.type) {
        case "stream_chunk":
          setStreamingContent(prev => prev + data.content)
          break
        case "token_update":
          // State auto-synced via agent.state — no manual handling needed
          break
        case "session_warning":
          setWarning(data)
          if (data.level === "95") {
            setShowExportModal(true)
          }
          break
        case "session_ended":
          setShowExportModal(true)
          break
        case "export_result":
          downloadExport(data.format, data.data)
          break
      }
    },
  })

  const sendMessage = (content: string) => {
    setStreamingContent("")
    agent.send(JSON.stringify({
      type: "chat_message",
      content,
    }))
  }

  const requestExport = (format: "md" | "json") => {
    agent.send(JSON.stringify({
      type: "export_session",
      format,
    }))
  }

  const requestEnd = () => {
    agent.send(JSON.stringify({ type: "end_session" }))
  }

  // Auto-scroll on new content
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" })
  }, [agent.state?.history?.length, streamingContent])

  if (!agent.state) {
    return (
      <div className="flex items-center justify-center h-full">
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary" />
      </div>
    )
  }

  const state = agent.state as SessionState

  return (
    <div className="flex flex-col h-full">
      <EphemeralBanner />

      {warning && <WarningBanner warning={warning} />}

      <div className="flex items-center justify-between px-4 py-2 border-b border-border">
        <div className="flex items-center gap-4">
          <span className="text-sm text-muted-foreground">
            {state.model}
          </span>
          <TokenCounter
            inputTokens={state.inputTokensUsed}
            outputTokens={state.outputTokensUsed}
            maxBudget={state.maxTokensBudget}
          />
        </div>
        <div className="flex gap-2">
          <Button
            variant="outline"
            size="sm"
            onClick={() => setShowExportModal(true)}
          >
            Export
          </Button>
          <Button
            variant="destructive"
            size="sm"
            onClick={requestEnd}
            disabled={state.status !== "active"}
          >
            End Session
          </Button>
        </div>
      </div>

      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        <ChatMessages
          messages={state.history}
          streamingContent={streamingContent}
        />
        <div ref={messagesEndRef} />
      </div>

      <ChatInput
        onSend={sendMessage}
        disabled={state.status !== "active"}
      />

      <ExportModal
        open={showExportModal}
        onOpenChange={setShowExportModal}
        onExport={requestExport}
        sessionEnded={state.status === "completed"}
      />
    </div>
  )
}
```

### 8.2 TokenCounter Component

File: `apps/user-application/src/components/session/token-counter.tsx`

```tsx
interface TokenCounterProps {
  inputTokens: number
  outputTokens: number
  maxBudget: number
}

export function TokenCounter({
  inputTokens,
  outputTokens,
  maxBudget,
}: TokenCounterProps) {
  const total = inputTokens + outputTokens
  const pct = maxBudget > 0 ? (total / maxBudget) * 100 : 0

  const colorClass =
    pct >= 95
      ? "text-destructive"
      : pct >= 80
        ? "text-yellow-500"
        : "text-muted-foreground"

  return (
    <div className="flex items-center gap-2 text-sm">
      <div className="flex items-center gap-1">
        <span className={colorClass}>
          {total.toLocaleString()} / {maxBudget.toLocaleString()}
        </span>
        <span className="text-muted-foreground">tokens</span>
      </div>
      <div className="w-24 h-2 bg-muted rounded-full overflow-hidden">
        <div
          className={`h-full rounded-full transition-all ${
            pct >= 95
              ? "bg-destructive"
              : pct >= 80
                ? "bg-yellow-500"
                : "bg-primary"
          }`}
          style={{ width: `${Math.min(pct, 100)}%` }}
        />
      </div>
    </div>
  )
}
```

### 8.3 EphemeralBanner Component

File: `apps/user-application/src/components/session/ephemeral-banner.tsx`

```tsx
import { AlertTriangle } from "lucide-react"

export function EphemeralBanner() {
  return (
    <div className="flex items-center gap-2 px-4 py-2 bg-yellow-500/10 border-b border-yellow-500/20">
      <AlertTriangle className="h-4 w-4 text-yellow-500 flex-shrink-0" />
      <span className="text-sm text-yellow-500">
        Conversation is NOT persisted. Save or export before ending the session.
      </span>
    </div>
  )
}
```

### 8.4 ExportModal Component

File: `apps/user-application/src/components/session/export-modal.tsx`

```tsx
import * as Dialog from "@radix-ui/react-dialog"
import { Button } from "@/components/ui/button"

interface ExportModalProps {
  open: boolean
  onOpenChange: (open: boolean) => void
  onExport: (format: "md" | "json") => void
  sessionEnded: boolean
}

export function ExportModal({
  open,
  onOpenChange,
  onExport,
  sessionEnded,
}: ExportModalProps) {
  return (
    <Dialog.Root open={open} onOpenChange={onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay className="fixed inset-0 bg-black/50" />
        <Dialog.Content className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-background text-foreground p-6 rounded-lg border w-full max-w-md">
          <Dialog.Title className="text-lg font-semibold">
            {sessionEnded ? "Session Ended" : "Export Conversation"}
          </Dialog.Title>
          <Dialog.Description className="text-sm text-muted-foreground mt-2">
            {sessionEnded
              ? "Your session has ended. Export your conversation before it is lost."
              : "Download your conversation in your preferred format."}
          </Dialog.Description>
          <div className="flex gap-3 mt-6">
            <Button
              variant="outline"
              className="flex-1"
              onClick={() => onExport("md")}
            >
              Export Markdown
            </Button>
            <Button
              variant="outline"
              className="flex-1"
              onClick={() => onExport("json")}
            >
              Export JSON
            </Button>
          </div>
          {!sessionEnded && (
            <Dialog.Close asChild>
              <Button variant="ghost" className="w-full mt-2">
                Cancel
              </Button>
            </Dialog.Close>
          )}
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  )
}
```

### 8.5 WarningBanner Component

File: `apps/user-application/src/components/session/warning-banner.tsx`

```tsx
interface WarningBannerProps {
  warning: {
    level: string
    message: string
    tokensRemaining: number
  }
}

export function WarningBanner({ warning }: WarningBannerProps) {
  const isCritical = warning.level === "95"
  return (
    <div
      className={`flex items-center gap-2 px-4 py-2 border-b ${
        isCritical
          ? "bg-destructive/10 border-destructive/20"
          : "bg-yellow-500/10 border-yellow-500/20"
      }`}
    >
      <span
        className={`text-sm font-medium ${
          isCritical ? "text-destructive" : "text-yellow-500"
        }`}
      >
        {warning.message} ({warning.tokensRemaining.toLocaleString()} tokens remaining)
      </span>
    </div>
  )
}
```

---

## 9. Wrangler Config Changes

File: `apps/data-service/wrangler.jsonc`

Add to each environment block (`dev`, `staging`, `production`):

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
  ]
}
```

Key: `new_sqlite_classes` (not `new_classes`) because Agents SDK uses SQLite for state persistence and scheduling.

### Env Type Additions

After running `pnpm run cf-typegen`, `Env` interface gains:

```ts
interface Env {
  // existing...
  SESSION_AGENT: DurableObjectNamespace
  PROVIDER_AGENT: DurableObjectNamespace
  POOL_CACHE: KVNamespace
  ANTHROPIC_API_KEY: string  // Worker secret
}
```

`ANTHROPIC_API_KEY` — set via `wrangler secret put ANTHROPIC_API_KEY` per environment. Used only for API-key-delegation mode when provider stores their key on platform. In Phase 2 MVP, this is the provider's encrypted key decrypted at runtime.

---

## 10. WebSocket Message Protocol

File: `packages/data-ops/src/zod-schema/ws-messages.ts`

### 10.1 Consumer -> SessionAgent

```ts
import { z } from "zod"

export const ChatMessageSchema = z.object({
  type: z.literal("chat_message"),
  content: z.string().min(1).max(100_000),
})

export const EndSessionMessageSchema = z.object({
  type: z.literal("end_session"),
})

export const ExportSessionMessageSchema = z.object({
  type: z.literal("export_session"),
  format: z.enum(["md", "json"]),
})

export const ConsumerMessageSchema = z.discriminatedUnion("type", [
  ChatMessageSchema,
  EndSessionMessageSchema,
  ExportSessionMessageSchema,
])

export type ConsumerMessage = z.infer<typeof ConsumerMessageSchema>
```

### 10.2 SessionAgent -> Consumer

```ts
export const StreamChunkMessageSchema = z.object({
  type: z.literal("stream_chunk"),
  content: z.string(),
})

export const TokenUpdateMessageSchema = z.object({
  type: z.literal("token_update"),
  inputTokensUsed: z.number(),
  outputTokensUsed: z.number(),
  maxTokensBudget: z.number(),
})

export const SessionWarningMessageSchema = z.object({
  type: z.literal("session_warning"),
  level: z.enum(["80", "95"]),
  message: z.string(),
  tokensRemaining: z.number(),
})

export const SessionEndedMessageSchema = z.object({
  type: z.literal("session_ended"),
  reason: z.enum([
    "tokens_exhausted",
    "time_window",
    "user_ended",
    "provider_revoked",
    "error",
  ]),
  creditsCharged: z.number(),
  creditsRefunded: z.number(),
})

export const ExportResultMessageSchema = z.object({
  type: z.literal("export_result"),
  format: z.enum(["md", "json"]),
  data: z.string(),
})

export const ErrorMessageSchema = z.object({
  type: z.literal("error"),
  message: z.string(),
})

export const AgentMessageSchema = z.discriminatedUnion("type", [
  StreamChunkMessageSchema,
  TokenUpdateMessageSchema,
  SessionWarningMessageSchema,
  SessionEndedMessageSchema,
  ExportResultMessageSchema,
  ErrorMessageSchema,
])

export type StreamChunkMessage = z.infer<typeof StreamChunkMessageSchema>
export type TokenUpdateMessage = z.infer<typeof TokenUpdateMessageSchema>
export type SessionWarningMessage = z.infer<typeof SessionWarningMessageSchema>
export type SessionEndedMessage = z.infer<typeof SessionEndedMessageSchema>
export type ExportResultMessage = z.infer<typeof ExportResultMessageSchema>
export type AgentMessage = z.infer<typeof AgentMessageSchema>
```

### 10.3 Session Zod Schemas

File: `packages/data-ops/src/zod-schema/session.ts`

```ts
import { z } from "zod"

export const SessionCreateRequestSchema = z.object({
  poolId: z.string().min(1),
  maxTokensBudget: z.number().int().min(1000).max(1_000_000),
  model: z.string().optional(),
})

export const SessionIdParamSchema = z.object({
  id: z.string().min(1),
})

export const SessionResponseSchema = z.object({
  id: z.string(),
  poolId: z.string(),
  consumerId: z.string(),
  providerId: z.string(),
  status: z.enum(["pending", "active", "ending", "completed", "aborted", "timeout"]),
  model: z.string(),
  inputTokensUsed: z.number(),
  outputTokensUsed: z.number(),
  maxTokensBudget: z.number(),
  creditsReserved: z.number(),
  creditsCharged: z.number(),
  endReason: z.enum([
    "tokens_exhausted",
    "time_window",
    "user_ended",
    "provider_revoked",
    "error",
  ]).nullable(),
  createdAt: z.string().datetime(),
})

export const SessionListQuerySchema = z.object({
  status: z.enum(["pending", "active", "ending", "completed", "aborted", "timeout"]).optional(),
  role: z.enum(["consumer", "provider"]).optional(), // filter by user's role in session
  limit: z.coerce.number().int().min(1).max(50).default(20),
  offset: z.coerce.number().int().min(0).default(0),
})

export type SessionCreateInput = z.infer<typeof SessionCreateRequestSchema>
export type SessionResponse = z.infer<typeof SessionResponseSchema>
export type SessionListQuery = z.infer<typeof SessionListQuerySchema>
```

---

## 11. Verification Checklist

### Session Lifecycle

- [ ] `POST /api/sessions` with valid `poolId` and sufficient credits returns session with `status: "pending"` and `creditsReserved > 0`
- [ ] `POST /api/sessions` with insufficient credits returns 400 `InsufficientCreditsError`
- [ ] `POST /api/sessions` with inactive pool returns 404
- [ ] `POST /api/sessions` when pool at max concurrent sessions returns 429
- [ ] Consumer credit_balance.available decreases by `creditsReserved`, credit_balance.reserved increases by same amount
- [ ] `GET /api/sessions/:id` returns session for owner, 403 for others

### WebSocket

- [ ] WS connection to `/ws/session/:sessionId` with valid auth cookie establishes connection
- [ ] WS connection without auth returns 401
- [ ] WS connection to session owned by another user returns 403
- [ ] On first WS connect, session transitions `pending` -> `active`
- [ ] Sending `chat_message` via WS returns `stream_chunk` messages followed by `token_update`

### LLM Proxy

- [ ] `streamLlmResponse` calls Claude API with correct headers and body
- [ ] `onChunk` callback fires for each `content_block_delta` SSE event
- [ ] `inputTokens` extracted from `message_start` event
- [ ] `outputTokens` extracted from `message_delta` event
- [ ] `LlmProxyError` thrown on non-200 API response with status code and body

### Token Metering

- [ ] `extractUsage` returns correct input/output/total from `LlmProxyResult`
- [ ] `calculateCreditsUsed` computes `ceil((in/1000)*rate + (out/1000)*rate)`
- [ ] Token counter in UI updates after each completion
- [ ] SessionAgent state `inputTokensUsed` and `outputTokensUsed` accumulate correctly

### Budget Enforcement

- [ ] At 80% usage, `session_warning` with `level: "80"` broadcast once
- [ ] At 95% usage, `session_warning` with `level: "95"` broadcast once
- [ ] At 100% usage, next `streamResponse` call ends session with `tokens_exhausted`
- [ ] Warning not re-sent if already in `warningsSent`

### Credit Settlement

- [ ] `POST /api/sessions/:id/end` triggers settlement
- [ ] Settlement: consumer `credit_balance.reserved` decreases by `creditsReserved`
- [ ] Settlement: consumer `credit_balance.available` increases by refund (reserved - actual)
- [ ] Settlement: provider `credit_balance.available` increases by `actualCredits`
- [ ] Consumer `credit_ledger` entry with type `consumer_spend` and negative amount
- [ ] Provider `credit_ledger` entry with type `provider_earning` and positive amount
- [ ] Session row updated with `status: "completed"`, final token counts, `creditsCharged`

### Session End & Export

- [ ] `session_ended` message broadcast with reason and credit info
- [ ] Export markdown: contains session ID, model, token counts, all messages
- [ ] Export JSON: valid JSON with messages array and metadata
- [ ] 30m idle timeout ends session with `reason: "timeout"`

### UI

- [ ] Ephemeral banner visible on session page
- [ ] Token counter shows progress bar with color changes at 80%/95%
- [ ] 80% warning displays yellow banner
- [ ] 95% warning displays red banner + opens export modal
- [ ] Export modal offers markdown and JSON download
- [ ] Session ended state shows export modal automatically
- [ ] Chat input disabled when session not active

### ProviderAgent (scaffolding)

- [ ] `isOnline()` returns current online state
- [ ] `getStats()` returns stats object
- [ ] `forwardLlmRequest()` throws "not implemented" error
- [ ] KV `provider:online:{id}` set to `"true"` with 120s TTL on CLI connect
- [ ] KV key deleted on CLI disconnect or heartbeat timeout

---

## 12. Unresolved Questions

- ~~How does provider's encrypted API key get decrypted at runtime?~~ **RESOLVED: api_key mode blocked until Phase 5. Phases 1-4 = local_proxy only.**
- ~~Should `syncCredits` schedule persist usage_log entries from DO SQLite to Postgres, or only on session end?~~ **RESOLVED: all fields sync every 60s (inputTokensUsed, outputTokensUsed). Full usage_log stays in DO SQLite.**
- ~~Max history length before conversation becomes too large for Claude context window? Truncation strategy?~~ **RESOLVED: sliding window — last 50 messages sent to Claude. Full history kept in state for export.**
- ~~`routeAgentRequest` vs custom Hono WS route for auth~~ **RESOLVED: custom Hono WS route. Auth before upgrade.**
- ~~Should credit settlement be a DB transaction?~~ **RESOLVED: yes, single `db.transaction()`.**
- ~~Token budget — `maxTokensBudget` input+output or output only?~~ **RESOLVED: output only.**
- ~~`useAgent` vs `useAgentChat`~~ **RESOLVED: `useAgent` directly. No custom wrapper — `onMessage` handler dispatches by type.**
- ~~Should the 30m idle timer reset on each message, or fire once and check last activity?~~ **RESOLVED: resets on each message (see `resetIdleTimer()` in 005).**
- ~~Export format: include system messages?~~ **RESOLVED: no. Only user/assistant messages in export. System prompt excluded.**

