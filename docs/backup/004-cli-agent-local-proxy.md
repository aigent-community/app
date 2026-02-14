# 004 - CLI Agent + Local Proxy

## Overview

Phase 4 adds `apps/cli/` -- a Node.js CLI tool (`tokswap`) that providers run locally. It connects to the platform's ProviderAgent via WebSocket, receives LLM requests from consumers, forwards them to the Claude API using the provider's local API key, and streams responses back. The provider's API key never leaves their machine.

## Context & Background

Phase 2 implemented the API-key-delegation proxy mode where the platform calls Claude directly using an encrypted provider key. Phase 4 adds the second proxy mode: **local proxy**. The CLI bridges the platform and the provider's local Claude API access. This is the privacy-first option -- keys stay local.

The critical design surface is the WS message protocol between CLI and ProviderAgent, which must handle multiplexed concurrent requests with streaming.

## Goals

- Provider can `tokswap start` and serve LLM requests via local Claude API key
- Multiple concurrent sessions routed through single WS connection
- Streaming responses with request ID multiplexing
- Auto-reconnect with exponential backoff
- Graceful shutdown preserving in-flight requests

## Non-Goals

- GUI for the CLI (terminal only)
- Support for LLM providers other than Claude (MVP)
- CLI auto-update mechanism
- Provider-side request filtering/moderation
- npm/global install -- MVP is workspace-only (`pnpm run dev:cli` / `node dist/index.js`)

---

## 1. CLI Package Setup

### Directory Structure

```
apps/cli/
  package.json
  tsconfig.json
  tsup.config.ts
  src/
    index.ts                    # commander entry
    commands/
      start.ts                  # connect + proxy loop
      status.ts                 # show connection/session info
      login.ts                  # store API token
      logout.ts                 # clear credentials
    proxy/
      ws-client.ts              # WS connection + reconnect + heartbeat
      llm-forwarder.ts          # Claude API streaming proxy
      message-router.ts         # dispatch incoming WS messages
    config/
      store.ts                  # ~/.tokswap/config read/write
      constants.ts              # defaults, paths
    types/
      messages.ts               # re-exports from @repo/data-ops
      state.ts                  # CLI internal state types
    utils/
      logger.ts                 # structured console output
```

### package.json

```jsonc
{
  "name": "@repo/cli",
  "version": "0.1.0",
  "private": true,
  "bin": {
    "tokswap": "./dist/index.js"
  },
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.39.0",
    "@repo/data-ops": "workspace:*",
    "commander": "^13.1.0",
    "ws": "^8.18.0"
  },
  "devDependencies": {
    "@types/ws": "^8.18.0",
    "@types/node": "^22.18.12",
    "tsup": "^8.5.0",
    "typescript": "^5.9.3"
  }
}
```

### tsup.config.ts

```ts
import { defineConfig } from "tsup"

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  target: "node20",
  outDir: "dist",
  clean: true,
  sourcemap: true,
  banner: { js: "#!/usr/bin/env node" },
  // externalize workspace dep -- resolved at runtime
  external: ["@repo/data-ops"],
})
```

### tsconfig.json

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "Bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "lib": ["ES2022"],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

### Root package.json additions

```jsonc
{
  "scripts": {
    "dev:cli": "pnpm run --filter @repo/cli dev",
    "build:cli": "pnpm run --filter @repo/cli build"
  }
}
```

pnpm-workspace.yaml already covers `apps/*`, so `apps/cli` is auto-discovered.

---

## 2. CLI Commands

### Entry Point -- `src/index.ts`

```ts
import { Command } from "commander"
import { startCommand } from "./commands/start"
import { statusCommand } from "./commands/status"
import { loginCommand } from "./commands/login"
import { logoutCommand } from "./commands/logout"

const program = new Command()
  .name("tokswap")
  .description("TokSwap provider CLI agent")
  .version("0.1.0")

program.addCommand(startCommand)
program.addCommand(statusCommand)
program.addCommand(loginCommand)
program.addCommand(logoutCommand)

program.parse()
```

### 2a. `tokswap login` -- `commands/login.ts`

```ts
import { Command } from "commander"
import { writeConfig, readConfig } from "../config/store"

export const loginCommand = new Command("login")
  .description("Authenticate with TokSwap platform")
  .requiredOption("-t, --token <token>", "API token from settings page")
  .option("-s, --server <url>", "Platform URL", "https://api.tokswap.io")
  .action(async (opts: { token: string; server: string }) => {
    const existing = readConfig()
    writeConfig({
      ...existing,
      apiToken: opts.token,
      serverUrl: opts.server,
    })
    console.log("Authenticated. Run `tokswap start` to connect.")
  })
```

### 2b. `tokswap logout` -- `commands/logout.ts`

```ts
import { Command } from "commander"
import { readConfig, writeConfig } from "../config/store"

export const logoutCommand = new Command("logout")
  .description("Clear stored credentials")
  .action(() => {
    const config = readConfig()
    writeConfig({ ...config, apiToken: undefined })
    console.log("Logged out.")
  })
```

### 2c. `tokswap start` -- `commands/start.ts`

Core command. Connects WS, runs proxy loop, handles signals.

```ts
import { Command } from "commander"
import { readConfig, writeStatus, clearStatus } from "../config/store"
import { WsClient } from "../proxy/ws-client"
import { LlmForwarder } from "../proxy/llm-forwarder"
import { MessageRouter } from "../proxy/message-router"

export const startCommand = new Command("start")
  .description("Connect to platform and start proxying LLM requests")
  .option("-k, --api-key <key>", "Claude API key (or ANTHROPIC_API_KEY env)")
  .option("-m, --model <model>", "Default model", "claude-sonnet-4-20250514")
  .action(async (opts: { apiKey?: string; model: string }) => {
    const config = readConfig()
    if (!config.apiToken) {
      console.error("Not authenticated. Run `tokswap login` first.")
      process.exit(1)
    }

    const anthropicKey = opts.apiKey ?? process.env.ANTHROPIC_API_KEY
    if (!anthropicKey) {
      console.error("Claude API key required. Use --api-key or ANTHROPIC_API_KEY env.")
      process.exit(1)
    }

    const forwarder = new LlmForwarder(anthropicKey, opts.model)
    const wsClient = new WsClient(config.serverUrl, config.apiToken)
    const router = new MessageRouter(wsClient, forwarder)
    const startedAt = new Date().toISOString()

    wsClient.onMessage((data) => router.handle(data))

    // Write status file every 10s so `tokswap status` can read it
    const statusInterval = setInterval(() => {
      writeStatus({
        connected: wsClient.state === "connected",
        serverUrl: config.serverUrl,
        uptimeSeconds: Math.floor((Date.now() - new Date(startedAt).getTime()) / 1000),
        activeSessions: forwarder.activeCount,
        tokensServedToday: forwarder.totalTokensServed,
        pid: process.pid,
        startedAt,
      })
    }, 10_000)

    const shutdown = async () => {
      console.log("\nShutting down...")
      clearInterval(statusInterval)
      await forwarder.abortAll()
      wsClient.close()
      clearStatus()
      process.exit(0)
    }

    process.on("SIGINT", shutdown)
    process.on("SIGTERM", shutdown)

    await wsClient.connect()
  })
```

### 2d. `tokswap status` -- `commands/status.ts`

Reads status from a local Unix socket or status file written by the running `start` process.

```ts
import { Command } from "commander"
import { readStatusFile } from "../config/store"

export const statusCommand = new Command("status")
  .description("Show connection status")
  .action(() => {
    const status = readStatusFile()
    if (!status) {
      console.log("Not running.")
      return
    }
    console.log(`Status:     ${status.connected ? "Connected" : "Disconnected"}`)
    console.log(`Server:     ${status.serverUrl}`)
    console.log(`Uptime:     ${status.uptimeSeconds}s`)
    console.log(`Sessions:   ${status.activeSessions}`)
    console.log(`Tokens:     ${status.tokensServedToday} today`)
  })
```

The `start` command writes a status file to `~/.tokswap/status.json` periodically. `status` reads it. Simple, no IPC needed.

---

## 3. WS Client -- `proxy/ws-client.ts`

### Connection State Machine

```
DISCONNECTED ──connect()──> CONNECTING ──onopen──> CONNECTED
     ^                          |                      |
     |                          v                      v
     +--------backoff----- RECONNECTING <---onclose----+
```

### Implementation

```ts
import WebSocket from "ws"

type ConnectionState = "disconnected" | "connecting" | "connected" | "reconnecting"

interface WsClientConfig {
  heartbeatIntervalMs: number    // 25_000
  heartbeatTimeoutMs: number     // 10_000
  reconnectBaseMs: number        // 1_000
  reconnectMaxMs: number         // 30_000
  reconnectMaxAttempts: number   // Infinity
}

const DEFAULTS: WsClientConfig = {
  heartbeatIntervalMs: 25_000,
  heartbeatTimeoutMs: 10_000,
  reconnectBaseMs: 1_000,
  reconnectMaxMs: 30_000,
  reconnectMaxAttempts: Infinity,
}

export class WsClient {
  private ws: WebSocket | null = null
  private state: ConnectionState = "disconnected"
  private reconnectAttempt = 0
  private heartbeatTimer: ReturnType<typeof setInterval> | null = null
  private pongTimer: ReturnType<typeof setTimeout> | null = null
  private messageHandler: ((data: string) => void) | null = null
  private config: WsClientConfig

  constructor(
    private serverUrl: string,
    private authToken: string,
    config?: Partial<WsClientConfig>,
  ) {
    this.config = { ...DEFAULTS, ...config }
  }

  async connect(): Promise<void> {
    this.state = "connecting"
    const wsUrl = this.serverUrl
      .replace(/^http/, "ws")
      .replace(/\/$/, "")
      + "/api/ws/provider"

    this.ws = new WebSocket(wsUrl, {
      headers: { Authorization: `Bearer ${this.authToken}` },
    })

    this.ws.on("open", () => {
      this.state = "connected"
      this.reconnectAttempt = 0
      this.startHeartbeat()
      console.log("Connected to platform.")
    })

    this.ws.on("message", (raw: Buffer) => {
      const data = raw.toString()
      // handle pong internally
      if (data === "__pong__") {
        this.clearPongTimeout()
        return
      }
      this.messageHandler?.(data)
    })

    this.ws.on("close", (code: number) => {
      this.stopHeartbeat()
      if (this.state !== "disconnected") {
        console.log(`Disconnected (code ${code}). Reconnecting...`)
        this.scheduleReconnect()
      }
    })

    this.ws.on("error", (err: Error) => {
      console.error("WS error:", err.message)
    })
  }

  send(data: string): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(data)
    }
  }

  onMessage(handler: (data: string) => void): void {
    this.messageHandler = handler
  }

  close(): void {
    this.state = "disconnected"
    this.stopHeartbeat()
    this.ws?.close(1000, "shutdown")
  }

  getState(): ConnectionState { return this.state }

  private startHeartbeat(): void {
    this.heartbeatTimer = setInterval(() => {
      this.send("__ping__")
      this.pongTimer = setTimeout(() => {
        console.log("Heartbeat timeout. Reconnecting...")
        this.ws?.terminate()
      }, this.config.heartbeatTimeoutMs)
    }, this.config.heartbeatIntervalMs)
  }

  private stopHeartbeat(): void {
    if (this.heartbeatTimer) clearInterval(this.heartbeatTimer)
    this.clearPongTimeout()
  }

  private clearPongTimeout(): void {
    if (this.pongTimer) clearTimeout(this.pongTimer)
    this.pongTimer = null
  }

  private scheduleReconnect(): void {
    this.state = "reconnecting"
    const delay = Math.min(
      this.config.reconnectBaseMs * 2 ** this.reconnectAttempt,
      this.config.reconnectMaxMs,
    )
    // jitter: +/- 20%
    const jitter = delay * (0.8 + Math.random() * 0.4)
    this.reconnectAttempt++
    console.log(`Reconnecting in ${Math.round(jitter)}ms (attempt ${this.reconnectAttempt})`)
    setTimeout(() => this.connect(), jitter)
  }
}
```

### Reconnect strategy

- Exponential backoff: `base * 2^attempt`, capped at `reconnectMaxMs`
- 20% jitter to avoid thundering herd
- Reset attempt counter on successful connect
- No max attempts by default (provider should stay connected)

### Heartbeat

- CLI sends `__ping__` every 25s
- ProviderAgent responds `__pong__`
- If no pong within 10s, terminate + reconnect
- ProviderAgent also sends `heartbeat_ping` every 30s; CLI responds `heartbeat_pong`

Both sides monitor liveness independently.

---

## 4. LLM Forwarder -- `proxy/llm-forwarder.ts`

Receives requests from platform, calls Claude API, streams chunks back.

```ts
import Anthropic from "@anthropic-ai/sdk"

interface ForwardResult {
  requestId: string
  stream: AsyncIterable<Anthropic.MessageStreamEvent>
  abort: () => void
}

export class LlmForwarder {
  private client: Anthropic
  private activeRequests = new Map<string, AbortController>()
  private readonly maxConcurrent: number

  constructor(
    apiKey: string,
    private defaultModel: string,
    maxConcurrent = 10,
  ) {
    this.client = new Anthropic({ apiKey })
    this.maxConcurrent = maxConcurrent
  }

  async forward(request: LlmRequest): Promise<ForwardResult> {
    if (this.activeRequests.size >= this.maxConcurrent) {
      throw new Error(`Concurrent request limit reached (${this.maxConcurrent})`)
    }

    const controller = new AbortController()
    this.activeRequests.set(request.requestId, controller)

    const stream = this.client.messages.stream({
      model: request.model ?? this.defaultModel,
      max_tokens: request.maxTokens,
      messages: request.messages,
      system: request.system,
    }, { signal: controller.signal })

    return {
      requestId: request.requestId,
      stream,
      abort: () => controller.abort(),
    }
  }

  cancelRequest(requestId: string): void {
    const controller = this.activeRequests.get(requestId)
    if (controller) {
      controller.abort()
      this.activeRequests.delete(requestId)
    }
  }

  completeRequest(requestId: string): void {
    this.activeRequests.delete(requestId)
  }

  async abortAll(): Promise<void> {
    for (const [id, controller] of this.activeRequests) {
      controller.abort()
    }
    this.activeRequests.clear()
  }

  get activeCount(): number {
    return this.activeRequests.size
  }
}
```

---

## 5. Message Router -- `proxy/message-router.ts`

Dispatches incoming WS messages to appropriate handlers.

```ts
import { WsMessageSchema } from "@repo/data-ops/zod-schema/ws-messages"
import type { WsClient } from "./ws-client"
import type { LlmForwarder } from "./llm-forwarder"
import type { LlmRequest, LlmResponseChunk, LlmResponseEnd, LlmError } from "@repo/data-ops/zod-schema/ws-messages"

export class MessageRouter {
  constructor(
    private ws: WsClient,
    private forwarder: LlmForwarder,
  ) {}

  async handle(raw: string): Promise<void> {
    let parsed: unknown
    try {
      parsed = JSON.parse(raw)
    } catch {
      console.error("Malformed JSON from platform:", raw.slice(0, 100))
      return
    }
    const result = WsMessageSchema.safeParse(parsed)
    if (!result.success) {
      console.error("Invalid message:", result.error.message)
      return
    }

    const msg = result.data
    switch (msg.type) {
      case "llm_request":
        await this.handleLlmRequest(msg)
        break
      case "cancel_request":
        this.forwarder.cancelRequest(msg.requestId)
        break
      case "heartbeat_ping":
        this.ws.send(JSON.stringify({ type: "heartbeat_pong" }))
        break
      default:
        console.warn("Unknown message type:", msg.type)
    }
  }

  private async handleLlmRequest(request: LlmRequest): Promise<void> {
    try {
      const { requestId, stream } = await this.forwarder.forward(request)

      for await (const event of stream) {
        if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
          const chunk: LlmResponseChunk = {
            type: "llm_response_chunk",
            requestId,
            delta: event.delta.text,
          }
          this.ws.send(JSON.stringify(chunk))
        }
      }

      // stream complete -- get final message for usage
      const finalMessage = await stream.finalMessage()
      const end: LlmResponseEnd = {
        type: "llm_response_end",
        requestId,
        usage: {
          inputTokens: finalMessage.usage.input_tokens,
          outputTokens: finalMessage.usage.output_tokens,
        },
        stopReason: finalMessage.stop_reason ?? "end_turn",
      }
      this.ws.send(JSON.stringify(end))
      this.forwarder.completeRequest(requestId)

    } catch (err) {
      const error: LlmError = {
        type: "llm_error",
        requestId: request.requestId,
        error: err instanceof Error ? err.message : "Unknown error",
        code: this.classifyError(err),
      }
      this.ws.send(JSON.stringify(error))
      this.forwarder.completeRequest(request.requestId)
    }
  }

  private classifyError(err: unknown): string {
    if (err instanceof Error) {
      if (err.message.includes("rate_limit")) return "rate_limit"
      if (err.message.includes("overloaded")) return "overloaded"
      if (err.message.includes("invalid_api_key")) return "auth_error"
      if (err.name === "AbortError") return "cancelled"
    }
    return "internal_error"
  }
}
```

---

## 6. WS Message Protocol -- CLI <-> ProviderAgent

All messages are JSON with a `type` discriminator. Zod schemas defined in `packages/data-ops/src/zod-schema/ws-messages.ts`.

### Message Types

| Direction | Type | Purpose |
|-----------|------|---------|
| Platform -> CLI | `llm_request` | Forward LLM request to Claude API |
| Platform -> CLI | `cancel_request` | Cancel in-flight request |
| Platform -> CLI | `heartbeat_ping` | Server-initiated liveness check |
| CLI -> Platform | `llm_response_chunk` | Streaming text delta |
| CLI -> Platform | `llm_response_end` | Stream complete + usage |
| CLI -> Platform | `llm_error` | Request failed |
| CLI -> Platform | `heartbeat_pong` | Response to server ping |
| Bidirectional | `__ping__` / `__pong__` | Raw string heartbeat (not JSON) |

### Zod Schemas -- `packages/data-ops/src/zod-schema/ws-messages.ts`

```ts
import { z } from "zod"

// --- Shared ---

const MessageRoleSchema = z.enum(["user", "assistant"])

const ContentBlockSchema = z.object({
  type: z.literal("text"),
  text: z.string(),
})

const ConversationMessageSchema = z.object({
  role: MessageRoleSchema,
  content: z.union([z.string(), z.array(ContentBlockSchema)]),
})

// --- Platform -> CLI ---

export const LlmRequestSchema = z.object({
  type: z.literal("llm_request"),
  requestId: z.string(),
  sessionId: z.string(),
  messages: z.array(ConversationMessageSchema),
  system: z.string().optional(),
  model: z.string().optional(),
  maxTokens: z.number().int().positive(),
  temperature: z.number().min(0).max(2).optional(),
})

export const CancelRequestSchema = z.object({
  type: z.literal("cancel_request"),
  requestId: z.string(),
})

export const HeartbeatPingSchema = z.object({
  type: z.literal("heartbeat_ping"),
  timestamp: z.number(),
})

// --- CLI -> Platform ---

export const LlmResponseChunkSchema = z.object({
  type: z.literal("llm_response_chunk"),
  requestId: z.string(),
  delta: z.string(),
})

export const LlmResponseEndSchema = z.object({
  type: z.literal("llm_response_end"),
  requestId: z.string(),
  usage: z.object({
    inputTokens: z.number().int().nonnegative(),
    outputTokens: z.number().int().nonnegative(),
  }),
  stopReason: z.string(),
})

export const LlmErrorSchema = z.object({
  type: z.literal("llm_error"),
  requestId: z.string(),
  error: z.string(),
  code: z.enum([
    "rate_limit",
    "overloaded",
    "auth_error",
    "cancelled",
    "timeout",
    "internal_error",
  ]),
})

export const HeartbeatPongSchema = z.object({
  type: z.literal("heartbeat_pong"),
})

// --- Discriminated union ---

export const PlatformToCliSchema = z.discriminatedUnion("type", [
  LlmRequestSchema,
  CancelRequestSchema,
  HeartbeatPingSchema,
])

export const CliToPlatformSchema = z.discriminatedUnion("type", [
  LlmResponseChunkSchema,
  LlmResponseEndSchema,
  LlmErrorSchema,
  HeartbeatPongSchema,
])

// Combined for generic parsing
export const WsMessageSchema = z.discriminatedUnion("type", [
  LlmRequestSchema,
  CancelRequestSchema,
  HeartbeatPingSchema,
  LlmResponseChunkSchema,
  LlmResponseEndSchema,
  LlmErrorSchema,
  HeartbeatPongSchema,
])

// --- Types ---

export type LlmRequest = z.infer<typeof LlmRequestSchema>
export type CancelRequest = z.infer<typeof CancelRequestSchema>
export type LlmResponseChunk = z.infer<typeof LlmResponseChunkSchema>
export type LlmResponseEnd = z.infer<typeof LlmResponseEndSchema>
export type LlmError = z.infer<typeof LlmErrorSchema>
export type ConversationMessage = z.infer<typeof ConversationMessageSchema>
export type PlatformToCli = z.infer<typeof PlatformToCliSchema>
export type CliToPlatform = z.infer<typeof CliToPlatformSchema>
export type WsMessage = z.infer<typeof WsMessageSchema>
```

### Example Message Flow

Single request lifecycle:

```
SessionAgent                  ProviderAgent                CLI
    |                              |                        |
    |-- forwardLlmRequest() ------>|                        |
    |                              |-- llm_request -------->|
    |                              |                        |-- Anthropic API call
    |                              |<-- llm_response_chunk -|
    |                              |<-- llm_response_chunk -|
    |                              |<-- llm_response_chunk -|
    |                              |<-- llm_response_end ---|
    |<-- streaming chunks ---------|                        |
    |<-- final usage + end --------|                        |
```

Concurrent requests distinguished by `requestId` (ULID generated by SessionAgent).

---

## 7. ProviderAgent Updates (Server-Side)

### Updated ProviderAgent -- `apps/data-service/src/agents/provider-agent.ts`

```ts
import { Agent, callable } from "agents"
import type { LlmRequest, LlmResponseChunk, LlmResponseEnd, LlmError } from "@repo/data-ops/zod-schema/ws-messages"
import { CliToPlatformSchema } from "@repo/data-ops/zod-schema/ws-messages"

interface ProviderState {
  online: boolean
  cliConnected: boolean
  activeRequestCount: number
  totalRequestsServed: number
  lastSeenAt: number
}

interface PendingRequest {
  requestId: string
  sessionId: string // used to route chunks back via getAgentByName
  timeoutTimer: ReturnType<typeof setTimeout>
  createdAt: number
}

export class ProviderAgent extends Agent<Env, ProviderState> {
  initialState: ProviderState = {
    online: false,
    cliConnected: false,
    activeRequestCount: 0,
    totalRequestsServed: 0,
    lastSeenAt: 0,
  }

  private cliConnection: WebSocket | null = null
  private pendingRequests = new Map<string, PendingRequest>()
  private readonly REQUEST_TIMEOUT_MS = 120_000

  async onStart(): Promise<void> {
    this.scheduleEvery(30, "heartbeat")
  }

  // --- WS lifecycle (CLI connects here) ---

  onConnect(connection: WebSocket, ctx: { request: Request }): void {
    const authHeader = ctx.request.headers.get("Authorization")
    if (!this.validateCliToken(authHeader)) {
      connection.close(4001, "Unauthorized")
      return
    }

    // only one CLI connection per provider
    if (this.cliConnection) {
      this.cliConnection.close(4002, "Replaced by new connection")
    }

    this.cliConnection = connection
    this.setState({
      ...this.state,
      online: true,
      cliConnected: true,
      lastSeenAt: Date.now(),
    })

    this.updateKvStatus(true)
  }

  onMessage(connection: WebSocket, message: string | ArrayBuffer): void {
    const raw = typeof message === "string" ? message : new TextDecoder().decode(message)

    // raw heartbeat
    if (raw === "__ping__") {
      connection.send("__pong__")
      this.setState({ ...this.state, lastSeenAt: Date.now() })
      return
    }

    let parsed: unknown
    try {
      parsed = JSON.parse(raw)
    } catch {
      console.error("Malformed JSON from CLI:", raw.slice(0, 100))
      return
    }
    const result = CliToPlatformSchema.safeParse(parsed)
    if (!result.success) return

    const msg = result.data
    switch (msg.type) {
      case "llm_response_chunk":
      case "llm_response_end":
      case "llm_error":
        this.routeResponse(msg)
        break
      case "heartbeat_pong":
        this.setState({ ...this.state, lastSeenAt: Date.now() })
        break
    }
  }

  onClose(connection: WebSocket, code: number, reason: string): void {
    this.cliConnection = null
    this.setState({
      ...this.state,
      online: false,
      cliConnected: false,
    })
    this.updateKvStatus(false)

    // fail all pending requests via reverse RPC
    for (const [id, pending] of this.pendingRequests) {
      clearTimeout(pending.timeoutTimer)
      const sessionAgent = await getAgentByName<SessionAgent>(
        this.env.SESSION_AGENT,
        pending.sessionId,
      )
      await sessionAgent.handleProviderChunk({
        type: "llm_error",
        requestId: id,
        error: "Provider disconnected",
        code: "internal_error",
      })
    }
    this.pendingRequests.clear()
    this.setState({ ...this.state, activeRequestCount: 0 })
  }

  // --- Callable RPC (SessionAgent calls this) ---
  // @callable() uses JSON-serialized RPC — callbacks NOT supported.
  // Streaming: CLI sends chunks back via WS -> onMessage routes them ->
  // ProviderAgent calls SessionAgent.handleProviderChunk() via getAgentByName().

  @callable()
  async forwardLlmRequest(
    request: LlmRequest & { sessionId: string },
  ): Promise<void> {
    if (!this.cliConnection || !this.state.cliConnected) {
      // Reverse RPC: notify SessionAgent of error
      const sessionAgent = await getAgentByName<SessionAgent>(
        this.env.SESSION_AGENT,
        request.sessionId,
      )
      await sessionAgent.handleProviderChunk({
        type: "llm_error",
        requestId: request.requestId,
        error: "Provider offline",
        code: "internal_error",
      })
      return
    }

    // register pending request (maps requestId -> sessionId for routing chunks back)
    const timeoutTimer = setTimeout(() => {
      this.timeoutRequest(request.requestId)
    }, this.REQUEST_TIMEOUT_MS)

    this.pendingRequests.set(request.requestId, {
      requestId: request.requestId,
      sessionId: request.sessionId,
      timeoutTimer,
      createdAt: Date.now(),
    })

    this.setState({
      ...this.state,
      activeRequestCount: this.pendingRequests.size,
    })

    // forward to CLI
    this.cliConnection.send(JSON.stringify(request))
  }

  @callable()
  async isOnline(): Promise<boolean> {
    return this.state.online && this.state.cliConnected
  }

  @callable()
  async getStats(): Promise<{
    online: boolean
    activeRequests: number
    totalServed: number
  }> {
    return {
      online: this.state.online,
      activeRequests: this.pendingRequests.size,
      totalServed: this.state.totalRequestsServed,
    }
  }

  // --- Internal ---

  private async routeResponse(msg: LlmResponseChunk | LlmResponseEnd | LlmError): Promise<void> {
    const pending = this.pendingRequests.get(msg.requestId)
    if (!pending) return

    // Reverse RPC: route chunk back to SessionAgent
    const sessionAgent = await getAgentByName<SessionAgent>(
      this.env.SESSION_AGENT,
      pending.sessionId,
    )
    await sessionAgent.handleProviderChunk(msg)

    if (msg.type === "llm_response_end" || msg.type === "llm_error") {
      clearTimeout(pending.timeoutTimer)
      this.pendingRequests.delete(msg.requestId)
      this.setState({
        ...this.state,
        activeRequestCount: this.pendingRequests.size,
        totalRequestsServed: this.state.totalRequestsServed + 1,
      })
    }
  }

  private async timeoutRequest(requestId: string): Promise<void> {
    const pending = this.pendingRequests.get(requestId)
    if (!pending) return

    const sessionAgent = await getAgentByName<SessionAgent>(
      this.env.SESSION_AGENT,
      pending.sessionId,
    )
    await sessionAgent.handleProviderChunk({
      type: "llm_error",
      requestId,
      error: "Request timed out (120s)",
      code: "timeout",
    })
    this.pendingRequests.delete(requestId)
    this.setState({
      ...this.state,
      activeRequestCount: this.pendingRequests.size,
    })

    // send cancel to CLI so it aborts the API call
    this.cliConnection?.send(JSON.stringify({
      type: "cancel_request",
      requestId,
    }))
  }

  private async updateKvStatus(online: boolean): Promise<void> {
    await this.env.POOL_CACHE.put(
      `provider:status:${this.name}`,
      JSON.stringify({ online, lastSeenAt: Date.now() }),
      { expirationTtl: 300 },
    )
  }

  private validateCliToken(authHeader: string | null): boolean {
    if (!authHeader?.startsWith("Bearer ")) return false
    const token = authHeader.slice(7)
    // validated against DB-stored hashed token
    // implementation depends on api_token table (see section 8)
    return token.length > 0
  }

  // scheduled heartbeat
  async heartbeat(): Promise<void> {
    if (!this.cliConnection) return
    this.cliConnection.send(JSON.stringify({
      type: "heartbeat_ping",
      timestamp: Date.now(),
    }))
  }
}
```

### Request Flow: SessionAgent -> ProviderAgent -> CLI

When SessionAgent needs an LLM response in local-proxy mode:

```ts
// Inside SessionAgent.streamResponse()
const providerAgent = await getAgentByName<ProviderAgent>(
  this.env.PROVIDER_AGENT,
  this.state.providerId,
)

// fire-and-forget: ProviderAgent forwards to CLI, then calls back via
// sessionAgent.handleProviderChunk() when chunks arrive (reverse RPC)
await providerAgent.forwardLlmRequest({
  ...llmRequest,
  sessionId: this.state.sessionId, // so ProviderAgent can route chunks back
})

// SessionAgent must implement handleProviderChunk():
@callable()
async handleProviderChunk(msg: LlmResponseChunk | LlmResponseEnd | LlmError): Promise<void> {
  switch (msg.type) {
    case "llm_response_chunk":
      this.broadcast(JSON.stringify({ type: "stream_chunk", delta: msg.delta }))
      break
    case "llm_response_end":
      this.handleStreamEnd(msg.usage)
      break
    case "llm_error":
      this.broadcast(JSON.stringify({ type: "stream_error", error: msg.error }))
      break
  }
}
```

### Concurrency Model

- Single WS connection per provider (CLI -> ProviderAgent)
- Multiple `forwardLlmRequest()` calls create multiple entries in `pendingRequests` map
- Each request has unique `requestId` -- CLI processes them concurrently
- Responses routed back by `requestId` to correct SessionAgent via `getAgentByName()` reverse RPC
- No head-of-line blocking: chunks from different requests interleave freely

---

## 8. Auth Token System

### Database Table -- `api_token`

Add to `packages/data-ops/src/drizzle/schema.ts`:

```ts
export const apiTokens = pgTable("api_token", {
  id: text("id").primaryKey(),          // ulid
  userId: text("user_id").notNull().references(() => auth_user.id, { onDelete: "cascade" }),
  name: text("name").notNull(),         // "My CLI token"
  tokenHash: text("token_hash").notNull().unique(),
  tokenPrefix: text("token_prefix").notNull(),  // "tks_...abc" (first 8 chars for display)
  lastUsedAt: timestamp("last_used_at"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  revokedAt: timestamp("revoked_at"),
})
```

### Token Generation Flow

1. User clicks "Generate Token" on settings page
2. Server generates cryptographically random token: `tks_<32 random hex chars>`
3. Server stores SHA-256 hash of token in DB
4. Token returned to user **once** -- never stored in plaintext
5. User copies token and runs `tokswap login --token tks_...`

### Token Validation (on WS connect)

```ts
// In ProviderAgent or WS upgrade handler
async function validateApiToken(token: string, env: Env): Promise<string | null> {
  const hash = await sha256(token)
  const record = await getApiTokenByHash(hash)
  if (!record || record.revokedAt) return null
  // update last_used_at async
  await updateApiTokenLastUsed(record.id)
  return record.userId
}
```

### API Endpoints

```
POST   /api/provider/tokens         -- generate new token
GET    /api/provider/tokens         -- list tokens (prefix + metadata, never full token)
DELETE /api/provider/tokens/:id     -- revoke token
```

### Zod Schemas

```ts
export const ApiTokenCreateRequestSchema = z.object({
  name: z.string().min(1).max(50),
})

export const ApiTokenResponseSchema = z.object({
  id: z.string(),
  name: z.string(),
  tokenPrefix: z.string(),
  lastUsedAt: z.string().nullable(),
  createdAt: z.string(),
})

export const ApiTokenCreatedResponseSchema = ApiTokenResponseSchema.extend({
  token: z.string(),  // full token, shown once
})
```

---

## 9. Local Config -- `~/.tokswap/config`

### Structure

```ts
interface CliConfig {
  apiToken?: string
  serverUrl: string
  defaultModel: string
}

// defaults
const DEFAULT_CONFIG: CliConfig = {
  serverUrl: "https://api.tokswap.io",
  defaultModel: "claude-sonnet-4-20250514",
}
```

### Status File

```ts
interface CliStatus {
  connected: boolean
  serverUrl: string
  uptimeSeconds: number
  activeSessions: number
  tokensServedToday: number
  pid: number
  startedAt: string
}
```

### Implementation -- `config/store.ts`

```ts
import fs from "node:fs"
import path from "node:path"
import os from "node:os"

const CONFIG_DIR = path.join(os.homedir(), ".tokswap")
const CONFIG_PATH = path.join(CONFIG_DIR, "config.json")
const STATUS_PATH = path.join(CONFIG_DIR, "status.json")

function ensureDir(): void {
  if (!fs.existsSync(CONFIG_DIR)) {
    fs.mkdirSync(CONFIG_DIR, { mode: 0o700 })
  }
}

export function readConfig(): CliConfig {
  ensureDir()
  if (!fs.existsSync(CONFIG_PATH)) return { ...DEFAULT_CONFIG }
  const raw = fs.readFileSync(CONFIG_PATH, "utf-8")
  return { ...DEFAULT_CONFIG, ...JSON.parse(raw) }
}

export function writeConfig(config: CliConfig): void {
  ensureDir()
  fs.writeFileSync(CONFIG_PATH, JSON.stringify(config, null, 2), { mode: 0o600 })
}

export function writeStatus(status: CliStatus): void {
  ensureDir()
  fs.writeFileSync(STATUS_PATH, JSON.stringify(status, null, 2))
}

export function readStatusFile(): CliStatus | null {
  if (!fs.existsSync(STATUS_PATH)) return null
  try {
    const raw = fs.readFileSync(STATUS_PATH, "utf-8")
    const status = JSON.parse(raw) as CliStatus
    // stale if PID dead
    try { process.kill(status.pid, 0) } catch { return null }
    return status
  } catch {
    return null
  }
}

export function clearStatus(): void {
  if (fs.existsSync(STATUS_PATH)) fs.unlinkSync(STATUS_PATH)
}
```

File permissions: config dir `0o700`, config file `0o600` (user-only read/write since it contains the API token).

---

## 10. Settings Page -- `_auth/settings/api-keys.tsx`

### Route

`apps/user-application/src/routes/_auth/settings/api-keys.tsx`

### Behavior

1. **List tokens** -- table showing name, prefix (`tks_...abc`), created date, last used
2. **Generate** -- dialog with name input, shows full token once with copy button, warning that it won't be shown again
3. **Revoke** -- confirmation dialog, soft-delete (sets `revokedAt`)
4. **Limit** -- max 5 active tokens per user

### Server Functions

```ts
export const getApiTokens = createServerFn({ method: "GET" })
  .handler(async () => {
    // fetch from data-ops query
  })

export const createApiToken = createServerFn({ method: "POST" })
  .validator(ApiTokenCreateRequestSchema)
  .handler(async ({ data }) => {
    // generate token, hash, store, return with full token
  })

export const revokeApiToken = createServerFn({ method: "POST" })
  .validator(z.object({ id: z.string() }))
  .handler(async ({ data }) => {
    // set revokedAt
  })
```

---

## 11. Wrangler Config Updates

Add to `apps/data-service/wrangler.jsonc` (applies to all envs):

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
  ]
}
```

Note: `new_sqlite_classes` (not `new_classes`) per agents-sdk rule. The agents SDK requires SQLite-backed DOs for state sync.

---

## 12. Verification Checklist

| # | Test | Pass criteria |
|---|------|---------------|
| 1 | `tokswap login --token tks_xxx` stores token | `~/.tokswap/config.json` contains `apiToken` field |
| 2 | `tokswap logout` clears token | `apiToken` field removed from config |
| 3 | `tokswap start` without login | Exits with "Not authenticated" error |
| 4 | `tokswap start` without ANTHROPIC_API_KEY | Exits with "Claude API key required" error |
| 5 | `tokswap start` connects WS | Console shows "Connected to platform", ProviderAgent state `online: true` |
| 6 | WS auto-reconnects after disconnect | Reconnects with backoff, attempt counter increments, resets on success |
| 7 | Heartbeat timeout triggers reconnect | If pong not received in 10s, WS terminates and reconnects |
| 8 | LLM request forwarded and streamed back | Platform sends `llm_request`, CLI calls Claude, chunks stream back as `llm_response_chunk`, ends with `llm_response_end` + usage |
| 9 | Multiple concurrent requests | 3 simultaneous requests with different `requestId`s all complete correctly with right routing |
| 10 | Request timeout at 120s | ProviderAgent sends `llm_error` with code `timeout`, then `cancel_request` to CLI |
| 11 | CLI disconnect mid-request | All pending requests get `llm_error` with "Provider disconnected", KV status set offline |
| 12 | `tokswap status` shows live data | Reads `~/.tokswap/status.json`, shows connection state, session count, tokens served |
| 13 | `tokswap status` when not running | Shows "Not running" (stale PID detection) |
| 14 | SIGINT graceful shutdown | Aborts in-flight requests, closes WS cleanly (code 1000), removes status file |
| 15 | API token generation | Settings page generates token, shows once, stores hash in DB |
| 16 | API token revocation | Revoked token rejected on WS connect |
| 17 | Invalid token rejected on WS | ProviderAgent closes connection with code 4001 |
| 18 | Config file permissions | `~/.tokswap/` is 700, `config.json` is 600 |
| 19 | Claude API auth error | `llm_error` with code `auth_error` sent back |
| 20 | Claude API rate limit | `llm_error` with code `rate_limit` sent back |

---

## Alternatives Considered

**gRPC instead of WS** -- gRPC-web would work but adds complexity (protobuf, code gen). WS is simpler for bidirectional streaming and already used by agents-sdk for consumer connections.

**HTTP long-polling** -- Simpler but no true bidirectional streaming. Would require separate endpoints for sending chunks back, adding latency per chunk.

**Separate status server (HTTP on localhost)** -- Considered for `tokswap status` instead of status file. Adds port management complexity. Status file with PID check is simpler and sufficient.

**Token stored in OS keychain** -- More secure than file but adds platform-specific dependencies (keytar). Config file with restrictive permissions is acceptable for MVP.

---

## Security Considerations

- API token is SHA-256 hashed before DB storage; plaintext never persisted server-side
- Config file at `~/.tokswap/config.json` with 0600 permissions
- WS connection authenticated on upgrade via Bearer token
- Provider's Claude API key never transmitted to platform -- stays in CLI process memory
- ProviderAgent validates token against DB on every WS connect (not cached)
- One CLI connection per provider; new connection replaces old (prevents stale sessions)

## Performance Considerations

- Single WS connection multiplexes all requests -- no per-session connection overhead
- Streaming chunks forwarded immediately (no buffering) -- latency = WS round-trip only
- Claude API calls use `AbortController` for clean cancellation on timeout/disconnect
- Heartbeat interval (25s CLI / 30s server) keeps connection alive through proxies without excessive traffic
- Status file written every 10s, not per-request

---

## Resolved Questions

1. ~~`@callable()` streaming RPC -- callback functions?~~ **RESOLVED: NO. `@callable()` uses JSON-serialized RPC. Use reverse RPC via `getAgentByName()` — ProviderAgent calls `sessionAgent.handleProviderChunk()` per chunk.**
2. ~~Max concurrent requests per provider?~~ **RESOLVED: 10 concurrent (configurable via `LlmForwarder` constructor). Guard in `forward()`.**
3. ~~Token prefix format -- `tks_` or match existing Better Auth patterns?~~ **RESOLVED: `tks_` — distinct from auth tokens.**
4. ~~Should CLI support `--daemon` mode?~~ **RESOLVED: terminal-only for MVP. Workspace-only distribution.**
5. ~~Rate limit on WS messages from CLI to prevent abuse?~~ **RESOLVED: not MVP — trusted provider connection.**
6. ~~Should `cancel_request` be sent to CLI when consumer disconnects mid-stream?~~ **RESOLVED: yes — already implemented in `timeoutRequest`.**
7. ~~tsup or esbuild for CLI build?~~ **RESOLVED: tsup — already in package.json, DX worth the dep.**
8. ~~Should `@repo/data-ops` be bundled or resolved at runtime?~~ **RESOLVED: externalized via tsup config, resolved at runtime via workspace link.**
9. ~~`ConversationMessageSchema` — support image/tool_use blocks?~~ **RESOLVED: text-only for MVP. `content: z.string()` only.**
10. ~~`sessionId` in `LlmRequestSchema` — remove or keep?~~ **RESOLVED: keep. Used by ProviderAgent to route chunks back to correct SessionAgent via `getAgentByName()`.**

