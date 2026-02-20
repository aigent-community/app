# Phase 5 — Iteration (Usage-Driven)

## Overview

Post-launch improvements built only when users ask for them. Every feature has a trigger condition. Nothing ships preemptively.

Depends on: Phase 1-4 complete + real users.

## Goals

- Improve features users actually complain about
- Zero speculative work
- Each item has a "build when" trigger

## Non-Goals

- No new MCP tools
- No UI redesign
- No billing changes
- No premature instrumentation (no `access_count`, no analytics)

---

## Feature Catalog

### 1. Full-Text Search Upgrade (KEEP)

**Trigger**: users complain ILIKE search is too slow or returns bad results.

Phase 2 ships with ILIKE `%query%` on `activity_log.summary` and `shared_memory.content`. Upgrade to Postgres tsvector when quality/perf becomes a problem.

#### Schema Migration

```ts
// packages/data-ops/src/drizzle/schema/activities.ts
searchVector: text('search_vector') // populated via trigger
```

```sql
-- GIN index
CREATE INDEX activity_search_idx ON activity_log USING gin(search_vector);

-- Auto-update trigger
CREATE OR REPLACE FUNCTION activity_search_vector_update() RETURNS trigger AS $$
BEGIN
  NEW.search_vector := to_tsvector('english',
    coalesce(NEW.summary, '') || ' ' ||
    coalesce(array_to_string(
      ARRAY(SELECT jsonb_array_elements_text(NEW.tags)), ' '
    ), '')
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER activity_search_update
  BEFORE INSERT OR UPDATE ON activity_log
  FOR EACH ROW EXECUTE FUNCTION activity_search_vector_update();
```

Same pattern for `shared_memory.content` if memory search also degrades.

#### Query

```ts
function searchActivities(db: Database, orgId: string, query: string, limit: number, offset: number) {
  return db.select({
    ...activityLogColumns,
    rank: sql<number>`ts_rank(search_vector, plainto_tsquery('english', ${query}))`,
  })
  .from(activityLog)
  .where(and(
    eq(activityLog.orgId, orgId),
    sql`search_vector @@ plainto_tsquery('english', ${query})`
  ))
  .orderBy(sql`ts_rank(search_vector, plainto_tsquery('english', ${query})) DESC`)
  .limit(limit)
  .offset(offset)
}
```

#### Backfill

Migration must UPDATE existing rows to populate `search_vector`.

#### Files

| File | What |
|------|------|
| `packages/data-ops/src/drizzle/schema/activities.ts` | `search_vector` column |
| `packages/data-ops/src/drizzle/migrations/XXXX_activity_search.sql` | tsvector trigger + GIN index + backfill |
| `packages/data-ops/src/queries/activity-queries.ts` | `searchActivities()` replaces ILIKE version |
| `apps/data-service/src/hono/handlers/activity-handlers.ts` | Wire `?q=` to tsvector query |

---

### 2. Self-Hostable Deployment Guide (KEEP)

**Trigger**: first user asks "can I run this on my own infra?"

Same codebase, own Cloudflare account + Neon DB.

#### Template: `self-host/wrangler.jsonc`

```jsonc
{
  "$schema": "../apps/data-service/node_modules/wrangler/config-schema.json",
  "name": "aigent-self-hosted",
  "main": "../apps/data-service/src/index.ts",
  "compatibility_date": "2025-04-01",
  "compatibility_flags": ["nodejs_compat"],
  "env": {
    "production": {
      "name": "aigent-self-hosted",
      "routes": [
        { "pattern": "aigent.YOUR_DOMAIN.com", "custom_domain": true }
      ]
    }
  }
}
```

#### Setup Steps (`self-host/README.md`)

1. Clone repo
2. `pnpm install && pnpm run setup`
3. Create Neon project, get `DATABASE_URL`
4. `cp self-host/wrangler.jsonc apps/data-service/wrangler.jsonc`
5. Edit: set domain
6. Set secrets: `wrangler secret put DATABASE_URL --env production` (+ auth secrets)
7. Run migrations: `pnpm run migrate:production`
8. `pnpm run deploy:production:data-service`
9. Same pattern for `apps/user-application`

#### Key Constraints

- `custom_domain: true` not routes with `zone_name`
- SSL/TLS must be Full or Full (strict)
- Never use "Redirect from HTTP to HTTPS" rule

#### Files

| File | What |
|------|------|
| `self-host/wrangler.jsonc` | Data-service template |
| `self-host/wrangler-frontend.jsonc` | User-application template |
| `self-host/README.md` | Step-by-step guide |
| `self-host/.dev.vars.example` | Env vars template |

---

### 3. Vision Versioning (ADD)

**Trigger**: users want to track who changed what in org vision docs, or want to revert.

PLAN.md already has `version` column on `vision_document`. Phase 2 may ship without version history UI. This adds: version history list, diff view between versions, revert button.

#### API

```
GET /api/orgs/:orgId/vision/:docId/history        # list versions
GET /api/orgs/:orgId/vision/:docId/diff?v1=3&v2=5  # compare two versions
POST /api/orgs/:orgId/vision/:docId/revert/:version # create new version from old one
```

#### Diff Response

```ts
interface VisionDiff {
  docId: string
  title: string
  v1: number
  v2: number
  hunks: DiffHunk[]
}

interface DiffHunk {
  type: 'added' | 'removed' | 'unchanged'
  content: string
  lineStart: number
  lineEnd: number
}
```

#### Diff Algorithm

Use `diff` npm package (~8KB). No custom diffing.

```ts
import { diffLines } from 'diff'

function computeDiff(oldText: string, newText: string): DiffHunk[] {
  const changes = diffLines(oldText, newText)
  let line = 1
  return changes.map(change => {
    const hunk: DiffHunk = {
      type: change.added ? 'added' : change.removed ? 'removed' : 'unchanged',
      content: change.value,
      lineStart: line,
      lineEnd: line + (change.count ?? 0) - 1,
    }
    if (!change.removed) line += change.count ?? 0
    return hunk
  })
}
```

#### Frontend

Version history sidebar on vision doc page. Select two versions to compare. Inline diff view (green = added, red = removed). Revert = create new version from selected old version.

#### Files

| File | What |
|------|------|
| `apps/data-service/src/hono/services/vision-service.ts` | `diffVersions()`, `revertToVersion()` |
| `apps/data-service/src/hono/handlers/vision-handlers.ts` | `/history`, `/diff`, `/revert` handlers |
| `packages/data-ops/src/zod-schema/vision.ts` | `visionDiffQuerySchema`, `visionDiffResponseSchema` |
| `apps/user-application/src/routes/_auth/org/$orgId/vision.tsx` | Version history + diff viewer |

---

### 4. Activity Filters (ADD)

**Trigger**: activity feed gets noisy (10+ members, or 50+ daily activities).

Phase 2 ships basic chronological feed. This adds filter controls.

#### Filters

| Filter | Param | Notes |
|--------|-------|-------|
| Date range | `from`, `to` | ISO timestamps |
| User | `userId` | single user filter |
| Tags | `tags` | comma-separated, match any |

#### API

```
GET /api/orgs/:orgId/activities?from=2026-01-01&to=2026-02-01&userId=xxx&tags=refactor,auth
```

#### Zod

```ts
export const activityFilterSchema = z.object({
  from: z.coerce.date().optional(),
  to: z.coerce.date().optional(),
  userId: z.string().optional(),
  tags: z.string().optional().transform(v => v?.split(',')),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  offset: z.coerce.number().int().min(0).default(0),
})
```

#### Query

Build dynamic `where` clauses using Drizzle `and()`. Tags filter: `sql\`tags ?| array[${tags}]\`` (jsonb `?|` operator).

#### Frontend

Filter bar above activity feed: date pickers, user dropdown, tag chips. Persist filters in URL search params.

#### Files

| File | What |
|------|------|
| `packages/data-ops/src/zod-schema/activity.ts` | `activityFilterSchema` |
| `packages/data-ops/src/queries/activity-queries.ts` | Dynamic filter builder |
| `apps/data-service/src/hono/handlers/activity-handlers.ts` | Parse filter params |
| `apps/user-application/src/routes/_auth/org/$orgId/dashboard.tsx` | Filter bar UI |

---

### 5. Memory Browser Enhancements (ADD)

**Trigger**: memory list becomes unwieldy (50+ entries), or users can't find relevant memories.

#### Category Filter

Already have `memoryCategoryEnum` (pattern / decision / convention / learning). Add category tabs/dropdown to memory browser page.

#### Relevance Scoring (Heuristic)

No embeddings. Simple score for sort order.

| Factor | Weight | Logic |
|--------|--------|-------|
| Recency | 0.4 | `1 / (1 + daysSinceCreation / 30)` |
| Category match | 0.3 | Exact match to filter = 1.0, else 0.0 |
| Source quality | 0.3 | Has `source_activity_id` (auto-extracted) = 1.0, manual = 0.5 |

No `access_count` column. Premature instrumentation. Recency + source quality is enough.

```ts
interface ScoredMemory {
  memory: SharedMemory
  score: number
}

function scoreMemory(memory: SharedMemory, categoryFilter?: string): number {
  const daysSince = (Date.now() - memory.createdAt.getTime()) / (1000 * 60 * 60 * 24)
  const recency = 1 / (1 + daysSince / 30)
  const categoryMatch = categoryFilter && memory.category === categoryFilter ? 1.0 : 0.0
  const sourceQuality = memory.sourceActivityId ? 1.0 : 0.5

  return recency * 0.4 + categoryMatch * 0.3 + sourceQuality * 0.3
}
```

MCP `get_shared_memory` returns results sorted by score when category param provided.

#### Files

| File | What |
|------|------|
| `apps/data-service/src/hono/services/memory-service.ts` | `scoreMemory()`, sorted query |
| `apps/user-application/src/routes/_auth/org/$orgId/memory.tsx` | Category tabs, sorted display |

---

## What Was Moved / Dropped

| Item | Disposition | Where |
|------|------------|-------|
| Rate limiting per token | Moved to Phase 2 | `002-*.md` |
| CLAUDE.md export | Moved to Phase 4 | `004-*.md` |
| Memory `access_count` | Dropped | Premature instrumentation |
| Vision diffing (standalone) | Dropped | Bundled into vision versioning above |

---

## Implementation Order

Only build what's triggered. But if building multiple:

1. **Activity filters** — small, no migration, high UX impact
2. **Full-text search upgrade** — migration required, do with activity filters if both triggered
3. **Vision versioning** — independent feature, schema already supports it
4. **Memory enhancements** — smallest scope, add when needed
5. **Self-host guide** — documentation, do when config is stable

---

## Testing

| Feature | Test Type | What |
|---------|-----------|------|
| Full-text search | Integration | Insert rows, verify tsvector query returns ranked results |
| Vision diffing | Unit | `computeDiff()` with known inputs |
| Activity filters | Integration | Verify combinations of date/user/tag filters |
| Memory scoring | Unit | `scoreMemory()` with fixed timestamps |
| Self-host | Manual | Deploy from template to fresh CF account |

---

## Unresolved Questions

- tsvector: `plainto_tsquery` vs `websearch_to_tsquery` (AND/OR/NOT support)?
- Vision versioning: cap max versions per doc, or unlimited?
- Activity filter tags: match ANY or ALL when multiple tags provided?
- Relevance scoring weights: hardcoded OK or make per-org configurable?
- Self-host: CF-only or include Docker Compose for non-CF users?
- Memory categories: allow custom categories per org, or keep fixed enum?

---