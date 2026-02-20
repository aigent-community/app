# Phase 2 -- MCP Server (the product)

## Overview

Remote MCP server inside data-service CF Worker. SSE transport. Five tools: org context, shared memory, decision search, activity reporting, recent activities. Auth via `at_` prefixed tokens (SHA-256 hashed, org-scoped). Basic rate limiting via KV counter per token.

MCP server IS the product -- no point building UI before agents can connect. Moved from Phase 3 to Phase 2.

## Goals

- Employees connect Claude Code to org context via `.mcp.json`
- All 5 MCP tools operational with Zod-validated I/O
- Token auth middleware validates token + org membership per request
- MCP setup page in user-application with copy-paste `.mcp.json` snippet
- Rate limiting per token (100 req/min via CF Workers KV)

## Non-Goals

- Memory auto-extraction from activities (Phase 4)
- Alignment alerts (Phase 4)
- Self-hosted deployment guide (Phase 5)
- Advanced rate limiting (sliding window, tiered limits) -- simple counter sufficient for now

## Architecture

```
Claude Code (employee machine)
  | SSE transport
  v
GET  /mcp/sse?org=slug&token=at_xxx    -> SSE stream (event source)
POST /mcp/messages                      -> JSON-RPC messages
  |
  v
Rate Limit Middleware (KV counter per token, 100 req/min)
  |
  v
MCP Auth Middleware
  +- extract token from query param (SSE) or header (messages)
  +- SHA-256 hash token -> lookup api_token table
  +- verify: is_active, not expired, org matches
  +- verify: user is active member of org
  |
  v
MCP Server (apps/data-service/src/mcp/server.ts)
  +- get_organization_context  -> vision-service -> data-ops queries
  +- get_shared_memory         -> memory-service -> data-ops queries
  +- search_decisions          -> memory-service -> data-ops queries
  +- report_activity           -> activity-service -> data-ops queries
  +- get_recent_activities     -> activity-service -> data-ops queries
```

Lives in same CF Worker as Hono API. MCP routes mounted alongside existing routes in `app.ts`.

## File Structure

```
apps/data-service/src/
  mcp/
    server.ts                    # MCP server init, SSE transport, tool registration
    middleware.ts                # token validation, org membership check
    rate-limit.ts                # KV-based rate limiter
    tools/
      context.ts                # get_organization_context
      memory.ts                 # get_shared_memory, search_decisions
      activity.ts               # report_activity, get_recent_activities
    types.ts                    # MCP-specific interfaces

packages/data-ops/src/
  zod-schema/
    mcp-messages.ts             # tool input/output Zod schemas
  queries/
    vision.ts                   # get active vision docs by org
    memory.ts                   # get memories, search memories
    activity.ts                 # insert activity, get recent activities
    api-token.ts                # lookup token by hash
    member.ts                   # check membership

apps/user-application/src/routes/
  _auth/settings/
    mcp-setup.tsx               # setup page w/ connection instructions
```

## MCP Server Implementation

### Transport: SSE

Using `@modelcontextprotocol/sdk` with SSE transport adapter for CF Workers.

```ts
// apps/data-service/src/mcp/server.ts
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js'
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js'

interface McpContext {
  orgId: string
  userId: string
  orgSlug: string
}

function createMcpServer(): McpServer {
  const server = new McpServer({
    name: 'aigent',
    version: '1.0.0',
  })

  registerContextTool(server)
  registerMemoryTools(server)
  registerActivityTools(server)

  return server
}
```

### Mounting in Hono

MCP routes mounted in `app.ts` alongside existing routes. SSE endpoint uses raw Request/Response (not Hono's helpers) because SSE transport needs direct stream control.

```ts
// apps/data-service/src/hono/app.ts (additions)
import { mcpSseHandler, mcpMessageHandler } from '@/mcp/server'
import { mcpAuthMiddleware } from '@/mcp/middleware'
import { mcpRateLimitMiddleware } from '@/mcp/rate-limit'

App.get('/mcp/sse', mcpRateLimitMiddleware(), mcpAuthMiddleware(), mcpSseHandler)
App.post('/mcp/messages', mcpMessageHandler)
```

SSE handler creates new `SSEServerTransport` per connection. Transport manages session; `/mcp/messages` routes JSON-RPC messages to correct session via session ID.

### Session Management

Each SSE connection = one session. Transport assigns session ID. POST `/mcp/messages` includes session ID to route to correct transport instance.

```ts
// Session store (in-memory, per Worker instance)
const sessions = new Map<string, SSEServerTransport>()

// SSE handler: create session, store transport
// Message handler: lookup session, forward message
```

Worker instances are ephemeral -- sessions don't survive Worker restarts. Claude Code reconnects automatically on disconnect.

## Rate Limiting

Public endpoint needs protection from day 1. Simple KV counter per token.

### Design

- **Storage**: CF Workers KV namespace `MCP_RATE_LIMIT`
- **Key**: `rl:{tokenHash}:{minuteBucket}` where minuteBucket = `Math.floor(Date.now() / 60000)`
- **Limit**: 100 requests per minute per token
- **TTL**: 120 seconds (auto-cleanup, covers current + previous minute)

### Implementation

```ts
// apps/data-service/src/mcp/rate-limit.ts
interface RateLimitConfig {
  maxRequests: number  // 100
  windowMs: number     // 60_000
}

async function checkRateLimit(
  kv: KVNamespace,
  tokenHash: string,
  config: RateLimitConfig = { maxRequests: 100, windowMs: 60_000 }
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const bucket = Math.floor(Date.now() / config.windowMs)
  const key = `rl:${tokenHash}:${bucket}`

  const current = parseInt(await kv.get(key) ?? '0', 10)

  if (current >= config.maxRequests) {
    return {
      allowed: false,
      remaining: 0,
      resetAt: (bucket + 1) * config.windowMs,
    }
  }

  // Increment (fire-and-forget is fine, slight over-count on race is acceptable)
  await kv.put(key, String(current + 1), { expirationTtl: 120 })

  return {
    allowed: true,
    remaining: config.maxRequests - current - 1,
    resetAt: (bucket + 1) * config.windowMs,
  }
}
```

### Wrangler Config

```jsonc
// apps/data-service/wrangler.jsonc (additions)
{
  "kv_namespaces": [
    { "binding": "MCP_RATE_LIMIT", "id": "..." }
  ]
}
```

### Rate Limit Response

When limit exceeded, return 429 before auth middleware runs:

```
HTTP 429 Too Many Requests
Retry-After: <seconds until reset>
{ "error": "rate limit exceeded", "retryAfter": 42 }
```

### Why KV, Not Durable Objects

KV is eventually consistent -- two concurrent requests might both read count=99 and both increment to 100, allowing 101 requests. Acceptable for a simple rate limiter. DO would be exact but overkill for 100 req/min soft limit.

## MCP Auth Middleware

### Token Format

Tokens prefixed `at_` followed by 32 random hex chars. Example: `at_a1b2c3d4e5f6...`

Generated client-side (Phase 1 token CRUD), only SHA-256 hash stored in DB.

### Validation Flow

```ts
// apps/data-service/src/mcp/middleware.ts

interface McpAuthResult {
  userId: string
  orgId: string
  orgSlug: string
  tokenId: string
}

async function validateMcpToken(
  token: string,
  orgSlug: string
): Promise<McpAuthResult> {
  // 1. SHA-256 hash the raw token
  const hash = await sha256(token)

  // 2. Lookup api_token by hash
  const apiToken = await findTokenByHash(hash)
  if (!apiToken || !apiToken.is_active) throw new McpAuthError('invalid token')

  // 3. Check expiry
  if (apiToken.expires_at && apiToken.expires_at < new Date()) {
    throw new McpAuthError('token expired')
  }

  // 4. Verify org matches slug
  const org = await findOrgBySlug(orgSlug)
  if (!org || org.id !== apiToken.org_id) throw new McpAuthError('org mismatch')

  // 5. Verify user is active member of org
  const member = await findActiveMember(apiToken.org_id, apiToken.user_id)
  if (!member) throw new McpAuthError('not a member')

  // 6. Update last_used_at (fire-and-forget)
  // ctx.waitUntil(updateTokenLastUsed(apiToken.id))

  return {
    userId: apiToken.user_id,
    orgId: apiToken.org_id,
    orgSlug: orgSlug,
    tokenId: apiToken.id,
  }
}
```

### SHA-256 Hashing

Uses Web Crypto API (available in CF Workers):

```ts
async function sha256(input: string): Promise<string> {
  const encoder = new TextEncoder()
  const data = encoder.encode(input)
  const hash = await crypto.subtle.digest('SHA-256', data)
  return Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('')
}
```

### Token Extraction

- SSE endpoint (`GET /mcp/sse`): token from `?token=at_xxx` query param
- Messages endpoint (`POST /mcp/messages`): session already authenticated (SSE created session with auth context)

Only SSE connection needs auth. Once connected, messages route to authenticated session.

### Error Responses

MCP auth errors return JSON (not SSE) with appropriate status:

| Error | Status | Body |
|-------|--------|------|
| missing token/org | 400 | `{ error: "missing token or org" }` |
| invalid token | 401 | `{ error: "invalid token" }` |
| token expired | 401 | `{ error: "token expired" }` |
| org mismatch | 403 | `{ error: "org mismatch" }` |
| not a member | 403 | `{ error: "not a member" }` |
| rate limit exceeded | 429 | `{ error: "rate limit exceeded", retryAfter: N }` |

## MCP Tool Schemas

### get_organization_context

Returns all active vision documents for the org.

```ts
// Input
z.object({})

// Output
z.object({
  orgName: z.string(),
  vision: z.array(z.object({
    id: z.string(),
    title: z.string(),
    content: z.string(),
    category: z.enum(['rules', 'decisions', 'context', 'goals']),
    version: z.number(),
    createdAt: z.string(),
  })),
})
```

Implementation: query `vision_document` where `org_id = ctx.orgId AND is_active = true`, ordered by category then title.

```ts
// apps/data-service/src/mcp/tools/context.ts
function registerContextTool(server: McpServer): void {
  server.tool(
    'get_organization_context',
    'Returns org vision, rules, decisions, and goals. Call at session start.',
    {},
    async (_args, extra) => {
      const ctx = getSessionContext(extra)
      const docs = await getActiveVisionDocs(ctx.orgId)
      const org = await getOrgById(ctx.orgId)
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({ orgName: org.name, vision: docs }),
        }],
      }
    }
  )
}
```

### get_shared_memory

Returns recent shared memory entries.

```ts
// Input
z.object({
  category: z.enum(['pattern', 'decision', 'convention', 'learning']).optional(),
  limit: z.number().int().min(1).max(100).default(20),
})

// Output
z.object({
  memories: z.array(z.object({
    id: z.string(),
    content: z.string(),
    category: z.enum(['pattern', 'decision', 'convention', 'learning']),
    createdAt: z.string(),
    createdBy: z.string(),
  })),
})
```

Query: `shared_memory` where `org_id = ctx.orgId AND is_active = true`, optional category filter, order by `created_at DESC`, limit.

### search_decisions

Full-text search over shared memory entries (decisions + patterns).

```ts
// Input
z.object({
  query: z.string().min(1).max(200),
})

// Output
z.object({
  results: z.array(z.object({
    id: z.string(),
    content: z.string(),
    category: z.enum(['pattern', 'decision', 'convention', 'learning']),
    createdAt: z.string(),
    createdBy: z.string(),
  })),
})
```

Implementation: `ILIKE '%query%'` on `content` column. Scope to `org_id = ctx.orgId AND is_active = true`. Limit 20.

Post-MVP: replace with pg_trgm or full-text search index.

### report_activity (mandatory)

Agent reports what it did.

```ts
// Input
z.object({
  summary: z.string().min(1).max(2000),
  filesChanged: z.array(z.string()).optional(),
  decisionsMade: z.array(z.string()).optional(),
  tags: z.array(z.string()).optional(),
})

// Output
z.object({
  id: z.string(),
  memoryExtracted: z.string().optional(),
})
```

Implementation:
1. Insert into `activity_log` with `org_id`, `user_id` from auth context
2. `session_id` passed via MCP session metadata (or omitted if unavailable)
3. `memoryExtracted` always `undefined` in Phase 2 -- auto-extraction is Phase 4

```ts
// apps/data-service/src/mcp/tools/activity.ts
server.tool(
  'report_activity',
  'Report what you did. MANDATORY: call at end of every task.',
  {
    summary: z.string().min(1).max(2000),
    filesChanged: z.array(z.string()).optional(),
    decisionsMade: z.array(z.string()).optional(),
    tags: z.array(z.string()).optional(),
  },
  async (args, extra) => {
    const ctx = getSessionContext(extra)
    const activity = await insertActivity({
      orgId: ctx.orgId,
      userId: ctx.userId,
      summary: args.summary,
      filesChanged: args.filesChanged ?? [],
      decisionsMade: args.decisionsMade ?? [],
      tags: args.tags ?? [],
    })
    return {
      content: [{
        type: 'text',
        text: JSON.stringify({ id: activity.id }),
      }],
    }
  }
)
```

### get_recent_activities

See what other agents in the org are doing.

```ts
// Input
z.object({
  limit: z.number().int().min(1).max(50).default(10),
  userId: z.string().optional(),
})

// Output
z.object({
  activities: z.array(z.object({
    id: z.string(),
    userId: z.string(),
    summary: z.string(),
    filesChanged: z.array(z.string()),
    decisionsMade: z.array(z.string()),
    tags: z.array(z.string()),
    createdAt: z.string(),
  })),
})
```

Query: `activity_log` where `org_id = ctx.orgId`, optional `user_id` filter, order by `created_at DESC`, limit.

## MCP Types

```ts
// apps/data-service/src/mcp/types.ts

interface McpSessionContext {
  orgId: string
  userId: string
  orgSlug: string
  tokenId: string
}

interface McpAuthError extends Error {
  code: 'INVALID_TOKEN' | 'TOKEN_EXPIRED' | 'ORG_MISMATCH' | 'NOT_MEMBER' | 'MISSING_PARAMS'
}

interface ActivityInput {
  orgId: string
  userId: string
  summary: string
  filesChanged: string[]
  decisionsMade: string[]
  tags: string[]
  sessionId?: string
}

interface VisionDocResult {
  id: string
  title: string
  content: string
  category: string
  version: number
  createdAt: string
}

interface MemoryResult {
  id: string
  content: string
  category: string
  createdAt: string
  createdBy: string
}

interface ActivityResult {
  id: string
  userId: string
  summary: string
  filesChanged: string[]
  decisionsMade: string[]
  tags: string[]
  createdAt: string
}
```

## Data-Ops Queries Needed

New query files in `packages/data-ops/src/queries/`:

| File | Functions |
|------|-----------|
| `vision.ts` | `getActiveVisionDocs(orgId)` |
| `memory.ts` | `getRecentMemories(orgId, category?, limit)`, `searchMemories(orgId, query)` |
| `activity.ts` | `insertActivity(input)`, `getRecentActivities(orgId, userId?, limit)` |
| `api-token.ts` | `findTokenByHash(hash)` |
| `member.ts` | `findActiveMember(orgId, userId)` |
| `organization.ts` | `findOrgBySlug(slug)`, `getOrgById(id)` |

All queries use Drizzle query builder, return typed results.

## MCP Setup Page

Route: `_auth/settings/mcp-setup.tsx`

### Content

1. **Org selector** -- dropdown of orgs user belongs to
2. **Token management** -- list existing tokens, create new, revoke
3. **Connection snippet** -- copy-paste `.mcp.json`:

```json
{
  "mcpServers": {
    "aigent": {
      "type": "url",
      "url": "https://api.aigent.community/mcp/sse?org={slug}&token={token}"
    }
  }
}
```

4. **Instructions**:
   - "Copy the snippet above into `.mcp.json` in your repo root (or `~/.claude/mcp.json` for global)"
   - "Restart Claude Code to pick up the new MCP server"
   - "Your agent will automatically receive org context on every session"

### Token Creation Flow

1. User clicks "Generate Token" for selected org
2. Client generates `at_` + 32 random hex chars
3. Client sends SHA-256 hash to `POST /api/tokens`
4. Server stores hash, returns token metadata
5. Client shows raw token **once** (never stored/retrievable again)
6. Token auto-populated into `.mcp.json` snippet

## Dependencies

| Package | Purpose |
|---------|---------|
| `@modelcontextprotocol/sdk` | MCP server SDK, SSE transport |

Add to `apps/data-service/package.json`.

KV namespace `MCP_RATE_LIMIT` must be created via wrangler for each environment.

## Implementation Order

1. Add `@modelcontextprotocol/sdk` to data-service
2. Create KV namespace `MCP_RATE_LIMIT` in wrangler.jsonc (dev/staging/prod)
3. Create `mcp/types.ts` with all interfaces
4. Create Zod schemas in `packages/data-ops/src/zod-schema/mcp-messages.ts`
5. Create data-ops queries (vision, memory, activity, api-token, member, organization)
6. Create `mcp/rate-limit.ts` -- KV counter per token
7. Create `mcp/middleware.ts` -- token validation + org membership
8. Create `mcp/tools/context.ts` -- `get_organization_context`
9. Create `mcp/tools/memory.ts` -- `get_shared_memory` + `search_decisions`
10. Create `mcp/tools/activity.ts` -- `report_activity` + `get_recent_activities`
11. Create `mcp/server.ts` -- SSE transport, tool registration, session management
12. Mount MCP routes in `app.ts`
13. Create MCP setup page in user-application
14. E2E test: connect Claude Code via `.mcp.json`, verify all 5 tools work

## Testing Strategy

- **Unit**: each tool handler with mocked data-ops queries
- **Integration**: MCP auth middleware + rate limiter with test tokens against real DB/KV
- **E2E**: connect MCP client (from SDK) to local dev server, exercise all tools
- **Manual**: connect Claude Code to local MCP server, verify tool discovery + execution

## Security Considerations

- Tokens hashed with SHA-256 before storage -- raw token never persisted
- Token scoped to one org -- cannot use across orgs
- Membership check on every SSE connection -- revoked members lose access immediately
- All tool inputs validated with Zod -- no raw user input hits queries
- SSE connections are per-Worker-instance -- no cross-instance session leakage
- `last_used_at` update is fire-and-forget (waitUntil) -- doesn't block tool execution
- Rate limiting from day 1 -- 100 req/min per token, prevents abuse on public endpoint

## Unresolved Questions

- SSE vs Streamable HTTP? SSE simpler, Streamable HTTP newer spec direction. SDK supports both
- Session context passing: thread `McpSessionContext` into tool handlers via SDK `extra` param or closure?
- `session_id` for `report_activity`: does Claude Code expose session identifier via MCP metadata?
- Token in query param (SSE URL) vs Authorization header? Query param leaks in logs but `.mcp.json` `url` field is only config option
- CORS for SSE endpoint: Claude Code connects directly (not browser), CORS irrelevant? Or browser-like HTTP client?
- Max concurrent SSE connections per Worker instance? Need to test CF Worker limits
- `search_decisions` -- `ILIKE` (simple, slow) or `pg_trgm` index now?
- Token rotation: support rotating tokens without disconnecting active sessions?
- KV rate limit race condition: acceptable to occasionally allow ~101-105 requests? Or need atomic increment via DO?
- Rate limit response: should MCP SDK handle 429 gracefully, or does Claude Code need special handling?

---