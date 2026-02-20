# aigent — Organizational Agent Context Layer

## Context

Shared context layer for dev teams using AI coding agents. Tech lead sets vision/rules, devs connect Claude Code via MCP, every agent gets org context and reports what it did.

Generic architecture — works for any org with MCP-capable agents, but v1 messaging targets engineering teams.

Built on existing `saas-on-cf` boilerplate (React 19 + TanStack Start, Hono on CF Workers, Drizzle + Neon Postgres, Better Auth).

## Architecture

```
Employee's Claude Code (BYOK — own API key)
  ↕ MCP protocol (stdio → remote)
Remote MCP Server (CF Worker)
  ├─ get_organization_context    → org vision, rules, decisions
  ├─ get_shared_memory           → org knowledge base
  ├─ report_activity             → mandatory: what agent did
  ├─ get_recent_activities       → what other agents are doing
  └─ search_decisions            → past decisions/patterns
  ↕ service binding
Hono API Worker
  ├─ /api/orgs                   → org CRUD
  ├─ /api/orgs/:id/vision        → version-controlled context docs
  ├─ /api/orgs/:id/activities    → activity feed
  ├─ /api/orgs/:id/memory        → shared knowledge base
  ├─ /api/orgs/:id/members       → member management
  └─ Neon PostgreSQL

TanStack Start SSR Worker ──service binding──► Hono API Worker
  └─ Dashboard (boss: vision editor, activity feed, members)
                (employee: org context, own activity, shared memory)

[Post-MVP] Cloudflare Sandbox (CF Containers)
  └─ Optional: run Claude Code in cloud container with org context pre-injected
```

### How it works

1. Boss creates org → writes vision (structured CLAUDE.md + rules + decisions)
2. Employee connects Claude Code via MCP server (`.mcp.json`)
3. On every session, agent calls `get_organization_context` → gets boss's vision
4. Agent calls `get_shared_memory` → gets org knowledge base
5. Agent works → **must** call `report_activity` with summary of what it did
6. Boss sees activity feed in dashboard → can flag misalignment
7. Learnings from one agent propagate to all via shared memory

### Employee setup

```json
// .mcp.json (in repo root, or ~/.claude/mcp.json for global)
{
  "mcpServers": {
    "aigent": {
      "type": "url",
      "url": "https://mcp.aigent.community/sse?org=acme&token=at_xxx"
    }
  }
}
```

Self-hosted alternative:
```json
{
  "mcpServers": {
    "aigent": {
      "type": "url",
      "url": "https://your-instance.example.com/sse?org=acme&token=at_xxx"
    }
  }
}
```

## Monorepo Structure

```
apps/
  data-service/src/
    mcp/
      server.ts               # remote MCP server (SSE transport)
      tools/
        context.ts            # get_organization_context tool
        memory.ts             # get_shared_memory, search_decisions tools
        activity.ts           # report_activity, get_recent_activities tools
    routes/
      orgs.ts                 # org CRUD
      vision.ts               # vision doc CRUD
      activities.ts           # activity feed
      memory.ts               # shared memory CRUD
      members.ts              # member management + invites
    services/
      vision-service.ts       # vision doc ops
      activity-service.ts     # activity ingestion + feed
      memory-service.ts       # knowledge base ops
      member-service.ts       # invite, roles, seat management

  user-application/src/routes/
    _auth/org/$orgId/
      dashboard.tsx           # admin: activity feed, alignment overview
      vision.tsx              # admin: CLAUDE.md editor
      members.tsx             # admin: invite, manage members
      memory.tsx              # shared knowledge base browser
      settings.tsx            # org settings, billing
    _auth/settings/
      mcp-setup.tsx           # MCP connection instructions + token

packages/
  data-ops/src/drizzle/schema/
    organizations.ts          # org, org_member
    vision.ts                 # vision docs
    activities.ts             # activity log
    memory.ts                 # shared memory entries
    api-tokens.ts             # MCP auth tokens
  data-ops/src/zod-schema/
    organization.ts
    vision.ts
    activity.ts
    memory.ts
    api-token.ts
    mcp-messages.ts           # MCP tool input/output schemas
```

## Database Schema

### organization
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| name | text | org display name |
| slug | text UNIQUE | URL-safe identifier |
| owner_id | text FK→user | creator/admin |
| max_seats | int | plan limit |
| is_active | boolean | |
| created_at | timestamp | |

### organization_member
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| org_id | text FK→organization | |
| user_id | text FK→user nullable | null until user signs up |
| email | text | invite target |
| role | enum | admin / member |
| joined_at | timestamp | |
| is_active | boolean | |

### vision_document
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| org_id | text FK→organization | |
| title | text | e.g. "Architecture Rules" |
| content | text | markdown content |
| category | enum | rules / decisions / context / goals |
| is_active | boolean | soft-delete |
| created_by | text FK→user | |
| created_at | timestamp | |
| updated_at | timestamp | |

### activity_log
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| org_id | text FK→organization | |
| user_id | text FK→user | which employee's agent |
| summary | text | what the agent did |
| files_changed | jsonb | string[] of file paths |
| decisions_made | jsonb | string[] of decisions |
| tags | jsonb | string[] categorization |
| session_id | text | Claude Code session identifier (opaque) |
| created_at | timestamp | |

### shared_memory
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| org_id | text FK→organization | |
| content | text | the knowledge/learning |
| category | text nullable | free-form, categories emerge from usage |
| source_activity_id | text FK→activity_log nullable | where this came from |
| created_by | text FK→user | |
| is_active | boolean | |
| created_at | timestamp | |

### api_token (MCP auth)
| Column | Type | Notes |
|--------|------|-------|
| id | text PK | ulid |
| user_id | text FK→user | |
| org_id | text FK→organization | scoped to one org |
| token_hash | text | SHA-256 hash, prefix `at_` |
| name | text | user label |
| last_used_at | timestamp nullable | |
| expires_at | timestamp nullable | |
| is_active | boolean | |
| created_at | timestamp | |

## Enums

```
orgMemberRoleEnum: admin | member
visionCategoryEnum: rules | decisions | context | goals
```

## API Endpoints

```
# Organizations
POST/GET        /api/orgs
GET/PATCH/DEL   /api/orgs/:id

# Members
POST            /api/orgs/:id/members/invite
GET             /api/orgs/:id/members
PATCH/DEL       /api/orgs/:id/members/:memberId

# Vision (context docs)
POST/GET        /api/orgs/:id/vision
GET/PATCH/DEL   /api/orgs/:id/vision/:docId

# Activities (agent reports)
POST            /api/orgs/:id/activities               # MCP server calls this
GET             /api/orgs/:id/activities                # feed with filters
GET             /api/orgs/:id/activities/:activityId

# Shared Memory
POST/GET        /api/orgs/:id/memory
PATCH/DEL       /api/orgs/:id/memory/:memoryId
GET             /api/orgs/:id/memory/search?q=          # text search

# API Tokens (MCP auth)
POST/GET        /api/tokens
DELETE          /api/tokens/:id

# MCP Server (SSE transport)
GET             /mcp/sse?org=slug&token=at_xxx          # SSE stream
POST            /mcp/messages                           # MCP JSON-RPC
```

## MCP Tools

### get_organization_context
Returns all active vision documents for the org. Agent should call this at session start.

Input: `{}`
Output: `{ vision: VisionDocument[], orgName: string }`

### get_shared_memory
Returns recent shared memory entries. Agent uses this to learn from org knowledge base.

Input: `{ category?: string, limit?: number }`
Output: `{ memories: SharedMemory[] }`

### search_decisions
Search past decisions and patterns.

Input: `{ query: string }`
Output: `{ results: SharedMemory[] }`

### report_activity (mandatory)
Agent reports what it did. Called at end of task or periodically.

Input: `{ summary: string, filesChanged?: string[], decisionsMade?: string[], tags?: string[] }`
Output: `{ id: string, memoryExtracted?: string }`

Server-side: activity is stored; all `decisionsMade[]` entries auto-extracted as shared_memory rows.

### get_recent_activities
See what other agents in the org are doing.

Input: `{ limit?: number, userId?: string }`
Output: `{ activities: Activity[] }`

## Security

- Better Auth session for all `/api/*` routes (Google OAuth added later)
- MCP server authenticates via `at_` prefixed API tokens (SHA-256 hashed in DB)
- Token scoped to specific org (user must be active member)
- All inputs validated with Zod
- Vision docs editable in-place (versioning deferred to Phase 5)
- Activity reporting mandatory — MCP server enforces via tool availability
- Rate limiting on MCP endpoints from Phase 2 (KV counter per token)

## Implementation Phases

### Phase 1 — Schema + Auth + Org (foundation)
- Drizzle schema + migrations for all tables
- Better Auth setup (email/password, session)
- Organization CRUD (create, read, update)
- Member management (invite by email → pending row, claimed on signup)
- API token CRUD (generate `at_` tokens, revoke, list)

### Phase 2 — MCP Server (the product)
- Remote MCP server on CF Workers (SSE transport)
- All 5 MCP tools
- MCP auth middleware (token validation, org membership check)
- KV-based rate limiting per token
- MCP setup page (connection instructions, `.mcp.json` snippet with token)

### Phase 3 — Vision + Activity UI
- Vision editor (simple textarea + save)
- Vision doc list page (by category)
- Activity feed (chronological, no filters)
- Vision + activity API endpoints

### Phase 4 — Dashboard + Memory (boss value)
- Single dashboard page: activity feed + member list + counts
- Memory auto-extraction from `decisionsMade[]` in activities
- Memory browser page with ILIKE search
- CLAUDE.md export (generate markdown from vision docs)

### Phase 5 — Iteration (build when users ask)
- Vision versioning + diff view
- Activity filters (date, user, tags)
- Full-text search (tsvector upgrade)
- Memory categories + relevance scoring
- Self-hostable deployment guide

### Phase 6 — Cloudflare Sandbox (post-MVP)
- CF Containers integration (blocked on GA)
- Run Claude Code in cloud container with org context pre-injected
- Web terminal UI, R2 persistent workspace

## Resolved Decisions

1. BYOK only — users bring their own Claude API key/subscription, platform never touches Anthropic API
2. MCP server hosted remotely on CF Workers (also self-hostable)
3. Activity reporting mandatory per org policy
4. Shared memory scoped per-org
5. Auth: Better Auth with email/password now, Google OAuth later
6. Pricing: per-seat
7. Vision docs edit in-place for v1 — versioning deferred to Phase 5
8. MCP transport: SSE (standard for remote MCP servers)
9. API tokens prefixed `at_`, stored as SHA-256 hash, scoped to one org
10. No Anthropic API proxying — zero legal risk
11. All `decisionsMade[]` auto-extracted as shared memory (Phase 4)
12. Self-hostable: same codebase, different wrangler config + own Neon DB
13. Rate limiting on MCP from Phase 2 — KV counter per token
14. Invite flow: pending member row with email, `userId` backfilled on signup
15. No `memoryCategoryEnum` — free-form text, categories emerge from usage
16. Positioning: dev teams as wedge (Claude Code), architecture org-agnostic — no dev-specific assumptions in schema/product
