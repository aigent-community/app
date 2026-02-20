# Phase 4 -- Dashboard + Memory + Export

## Overview

Single admin dashboard page (activity feed + member list + counts), memory auto-extraction from activities, memory browser with search, CLAUDE.md export button.

## Context

Phases 1-3 deliver schema, auth, orgs, vision store, activity ingestion, shared memory CRUD, and MCP server. Phase 4 adds boss-facing UI and server-side memory extraction.

## Goals

- Single dashboard page: recent activity feed + member list with activity counts
- Auto-extract ALL `decisionsMade[]` entries from `report_activity` into `shared_memory`
- Memory browser page with ILIKE search
- CLAUDE.md export: download org vision as markdown file

## Non-Goals

- Timeline visualization (separate routes)
- Per-member drill-down page
- Alignment flagging/enum on `activity_log`
- Billing/payments
- Full-text activity search

## Schema Changes

**None.** No new columns or enums on `activity_log`. Existing tables sufficient.

---

## Architecture

```
report_activity (MCP tool)
  │
  ▼
POST /api/orgs/:id/activities
  │
  ├─ activity-service.create()  → insert activity_log
  │
  └─ memory-extraction-service.extract()
       │
       └─ for each decisionsMade[] entry → insert shared_memory
           category: 'decision', source_activity_id: activity.id

Dashboard (single route)
  │
  ├─ GET /api/orgs/:id/activities          (recent, paginated)
  └─ GET /api/orgs/:id/members             (with activity counts)

Memory Browser (single route)
  │
  └─ GET /api/orgs/:id/memory/search?q=    (ILIKE search)

CLAUDE.md Export
  │
  └─ GET /api/orgs/:id/vision/export       (markdown file download)
```

---

## API Endpoints (New/Modified)

```
# Activity stats per member (enhanced existing)
GET  /api/orgs/:id/members
     → members[] with activityCount, lastActive fields

# Vision export (new)
GET  /api/orgs/:id/vision/export
     → Content-Disposition: attachment; filename="CLAUDE.md"
     → concatenated vision docs as markdown
```

Existing endpoints used as-is:
- `GET /api/orgs/:id/activities` (paginated feed)
- `GET /api/orgs/:id/memory/search?q=` (ILIKE search)

---

## Frontend Routes

### `dashboard.tsx`

Path: `/_auth/org/$orgId/dashboard`

Single page with two sections:
1. **Activity feed** -- recent activities, paginated. Each card: user, summary, files count, tags, timestamp
2. **Member sidebar/section** -- list of members with activity count + last active timestamp

SSR loader fetches both via service binding.

### `memory.tsx`

Path: `/_auth/org/$orgId/memory`

Moved from Phase 3 scope to Phase 4 implementation.

- Search input (debounced, hits `GET /api/orgs/:id/memory/search?q=`)
- List of memory entries: content, category badge, source activity link, created_at
- Admin: delete button per entry

### CLAUDE.md Export (button on vision page)

Add export button to existing `vision.tsx` page. Hits `GET /api/orgs/:id/vision/export`, triggers browser download of `CLAUDE.md` file.

---

## Memory Auto-Extraction

### Trigger

When `report_activity` is called with non-empty `decisionsMade[]`.

### Logic

No heuristic. Every entry in `decisionsMade[]` becomes a `shared_memory` row.

```
apps/data-service/src/hono/services/memory-extraction-service.ts
```

```ts
interface ExtractionResult {
  extracted: boolean
  memoryIds: string[]
}

async function extractMemories(
  db: DB,
  activity: ActivityLog
): Promise<ExtractionResult> {
  const decisions = activity.decisions_made as string[]
  if (!decisions?.length) return { extracted: false, memoryIds: [] }

  const ids: string[] = []
  for (const decision of decisions) {
    const memory = await memoryService.create(db, {
      orgId: activity.org_id,
      content: decision,
      category: 'decision',
      sourceActivityId: activity.id,
      createdBy: activity.user_id,
    })
    ids.push(memory.id)
  }

  return { extracted: true, memoryIds: ids }
}
```

### Integration

`activity-service.create()` calls `extractMemories()` after inserting activity. Returns `memoryExtracted: true` to MCP caller.

---

## Vision Export Endpoint

```
apps/data-service/src/hono/handlers/vision-handlers.ts
```

```ts
// GET /api/orgs/:id/vision/export
async function exportVision(c: Context) {
  const orgId = c.req.param('id')
  const docs = await visionService.getActiveByOrg(db, orgId)
  const org = await orgService.getById(db, orgId)

  let md = `# ${org.name}\n\n`
  for (const doc of docs) {
    md += `## ${doc.title}\n\n${doc.content}\n\n`
  }

  return c.body(md, 200, {
    'Content-Type': 'text/markdown',
    'Content-Disposition': `attachment; filename="CLAUDE.md"`,
  })
}
```

---

## Backend Services

### `memory-extraction-service.ts` -- New

- `extractMemories(db, activity)` -- iterate `decisionsMade[]`, insert each as `shared_memory`

### `activity-service.ts` -- Modified

- `create()` now calls `extractMemories()` after insert

### `vision-service.ts` -- Addition

- `exportAsMarkdown(db, orgId)` -- concatenate active vision docs

---

## Server Functions

```
apps/user-application/src/lib/server-fns/dashboard.ts
```

```ts
export const getDashboardData = createServerFn({ method: 'GET' })
  .validator(z.object({ orgId: z.string() }))
  .handler(async ({ data }) => {
    const [activities, members] = await Promise.all([
      fetchDataService(`/api/orgs/${data.orgId}/activities?limit=50`),
      fetchDataService(`/api/orgs/${data.orgId}/members`),
    ])
    return {
      activities: await activities.json(),
      members: await members.json(),
    }
  })

export const searchMemories = createServerFn({ method: 'GET' })
  .validator(z.object({ orgId: z.string(), q: z.string().optional() }))
  .handler(async ({ data }) => {
    const url = data.q
      ? `/api/orgs/${data.orgId}/memory/search?q=${encodeURIComponent(data.q)}`
      : `/api/orgs/${data.orgId}/memory`
    return (await fetchDataService(url)).json()
  })

export const exportVision = createServerFn({ method: 'GET' })
  .validator(z.object({ orgId: z.string() }))
  .handler(async ({ data }) => {
    const res = await fetchDataService(`/api/orgs/${data.orgId}/vision/export`)
    return res.text()
  })
```

---

## UI Components

All in `apps/user-application/src/components/dashboard/`:

| Component | Purpose |
|-----------|---------|
| `activity-feed.tsx` | Paginated activity list |
| `activity-card.tsx` | Single activity: user, summary, tags, timestamp |
| `member-list.tsx` | Members with activity counts + last active |

In `apps/user-application/src/components/memory/`:

| Component | Purpose |
|-----------|---------|
| `memory-list.tsx` | List of memory entries |
| `memory-search.tsx` | Debounced search input |
| `memory-card.tsx` | Single memory: content, category, source link |

---

## Query Keys

```ts
export const dashboardKeys = {
  all: ['dashboard'] as const,
  data: (orgId: string) => [...dashboardKeys.all, orgId] as const,
}

export const memoryKeys = {
  all: ['memory'] as const,
  list: (orgId: string) => [...memoryKeys.all, 'list', orgId] as const,
  search: (orgId: string, q: string) => [...memoryKeys.all, 'search', orgId, q] as const,
}
```

---

## Authorization

- Dashboard: any org member can view
- Memory browser: any org member can view, admin can delete
- Vision export: any org member
- Memory deletion: `role: 'admin'`

---

## Implementation Order

1. `memory-extraction-service.ts` + integrate into `activity-service.create()`
2. Vision export endpoint in `vision-handlers.ts`
3. Server functions: `dashboard.ts`
4. UI components: activity feed, member list
5. Dashboard route: `dashboard.tsx`
6. Memory components + `memory.tsx` route
7. Export button on `vision.tsx`

---

## Unresolved Questions

- Activity pagination: cursor-based or offset?
- Dashboard polling interval for refresh? or manual refresh only for v1?
- Memory search: ILIKE on `content` column sufficient or need `tsvector`?
- Export: include shared memories in CLAUDE.md or vision docs only?
- Should memory browser show source activity inline or link to it?

---

## Review Action Items (2026-02-20)

**P0 — Blockers**

- [ ] Fix `memoryExtracted` return shape — PLAN.md says `{ id, memoryExtracted?: string }`, service returns `{ extracted: boolean, memoryIds: string[] }`. Pick one, update both.
- [ ] Specify `GET /api/orgs/:id/members` SQL — define exact query for `activityCount` (COUNT activity_log per user_id) and `lastActive` (MAX created_at). Unimplementable without.
- [ ] Resolve memory soft-delete vs hard delete — `is_active` column exists → soft-delete implied. State explicitly.

**P1 — Needed before ship**

- [ ] Pagination spec for activity feed — pick cursor-based or offset, or explicitly state "no pagination in v1, limit=50"
- [ ] Clarify employee vs boss dashboard — PLAN.md shows two views, doc is uniform. State whether role-based views deferred to Phase 5.
- [ ] Consolidate `exportAsMarkdown` vs inline markdown in handler — pick one pattern
- [ ] Add `memoryKeys.list` query key for no-`q` fallback case

**P2 — Polish**

- [ ] Dashboard refresh — decide "manual refresh only for v1", add Refresh button to component spec
- [ ] Export content scope — decide whether shared_memory included in CLAUDE.md export
- [ ] Memory source activity link — specify if link goes to detail page or just shows activity ID

---