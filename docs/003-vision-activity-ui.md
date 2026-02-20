# Phase 3 -- Vision + Activity UI

## Overview

Admin-facing UI for vision document management and activity feed. By this phase, MCP server (Phase 2) is live -- real agent data flows in. This phase surfaces it.

Assumes Phase 1 (schema, auth, org, members, tokens) and Phase 2 (MCP server, vision/activity/memory API endpoints) complete.

## Scope

**In scope:**
- Vision doc list page (by category, inline edit or navigate to editor)
- Vision editor page (textarea + save, no live preview)
- Activity feed page (chronological, newest first, last 50 + load more)
- Zod schemas for vision/activity request/response
- API endpoints for vision CRUD + activity list
- Server functions (`createServerFn`) bridging UI to data-service

**Out of scope:**
- Memory browser UI (agents query via MCP, boss doesn't need UI yet -- Phase 4+)
- Live markdown preview (iteration)
- Activity filters: date/user/tag (iteration)
- Pagination with offset (use simple load-more)
- Version history drawer (no versioning in schema)
- Category tabs/filters on activity feed

---

## API Endpoints

All under `/api/orgs/:orgId/`. Auth: Better Auth session + org membership middleware.

### Vision

```
POST   /api/orgs/:orgId/vision              # create new vision doc
GET    /api/orgs/:orgId/vision              # list active docs (optional ?category= filter)
GET    /api/orgs/:orgId/vision/:docId       # get single doc
PATCH  /api/orgs/:orgId/vision/:docId       # update doc (overwrites content in place)
DELETE /api/orgs/:orgId/vision/:docId       # soft-delete (is_active=false)
```

No version history endpoint -- edits mutate the row directly.

### Activities

```
GET    /api/orgs/:orgId/activities          # feed (limit, cursor-based load-more)
GET    /api/orgs/:orgId/activities/:id      # single activity detail
```

POST handled by MCP server (Phase 2). UI is read-only for activities.

---

## Zod Schemas

### `packages/data-ops/src/zod-schema/vision.ts`

```ts
import { z } from "zod"

const visionCategoryValues = ["rules", "decisions", "context", "goals"] as const

export const VisionDocumentSchema = z.object({
  id: z.string(),
  orgId: z.string(),
  title: z.string(),
  content: z.string(),
  category: z.enum(visionCategoryValues),
  isActive: z.boolean(),
  createdBy: z.string(),
  createdAt: z.coerce.date(),
})

export const VisionCreateRequestSchema = z.object({
  title: z.string().min(1).max(200),
  content: z.string().min(1),
  category: z.enum(visionCategoryValues),
})

export const VisionUpdateRequestSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  content: z.string().min(1).optional(),
  category: z.enum(visionCategoryValues).optional(),
}).refine(d => d.title || d.content || d.category, {
  message: "At least one field required",
})

export const VisionListQuerySchema = z.object({
  category: z.enum(visionCategoryValues).optional(),
})

export type VisionDocument = z.infer<typeof VisionDocumentSchema>
export type VisionCreateInput = z.infer<typeof VisionCreateRequestSchema>
export type VisionUpdateInput = z.infer<typeof VisionUpdateRequestSchema>
export type VisionListQuery = z.infer<typeof VisionListQuerySchema>
```

### `packages/data-ops/src/zod-schema/activity.ts`

```ts
import { z } from "zod"

export const ActivityLogSchema = z.object({
  id: z.string(),
  orgId: z.string(),
  userId: z.string(),
  summary: z.string(),
  filesChanged: z.array(z.string()).nullable(),
  decisionsMade: z.array(z.string()).nullable(),
  tags: z.array(z.string()).nullable(),
  sessionId: z.string().nullable(),
  createdAt: z.coerce.date(),
})

export const ActivityFeedQuerySchema = z.object({
  limit: z.coerce.number().min(1).max(100).default(50),
  cursor: z.string().optional(), // ulid of last item for load-more
})

export type ActivityLog = z.infer<typeof ActivityLogSchema>
export type ActivityFeedQuery = z.infer<typeof ActivityFeedQuerySchema>
```

No `ActivityCreateRequestSchema` needed here -- activity creation is via MCP `report_activity` tool (Phase 2).

---

## File Structure

### data-ops (schemas + queries)

```
packages/data-ops/src/
  zod-schema/
    vision.ts                    # vision CRUD schemas
    activity.ts                  # activity feed schemas
  queries/
    vision.ts                    # vision DB operations
    activity.ts                  # activity feed query
```

### data-service (handlers + services)

```
apps/data-service/src/hono/
  handlers/
    vision-handlers.ts           # vision route handlers
    activity-handlers.ts         # activity route handlers
  services/
    vision-service.ts            # vision CRUD logic
    activity-service.ts          # feed query logic
```

### user-application (pages + server functions)

```
apps/user-application/src/
  routes/_auth/org/$orgId/
    vision.tsx                   # vision doc list + editor
    activities.tsx               # activity feed
  server/
    vision.ts                    # createServerFn wrappers for vision API
    activity.ts                  # createServerFn wrappers for activity API
```

---

## Query Layer

### `packages/data-ops/src/queries/vision.ts`

```ts
import { getDb } from "../database/setup"
import { visionDocument } from "../drizzle/schema"
import { eq, and, desc } from "drizzle-orm"
import { ulid } from "ulid"

export async function createVisionDoc(
  orgId: string, input: VisionCreateInput, userId: string
) {
  const db = getDb()
  const [result] = await db.insert(visionDocument).values({
    id: ulid(),
    orgId,
    version: 1, // always 1, no versioning
    title: input.title,
    content: input.content,
    category: input.category,
    createdBy: userId,
  }).returning()
  return result
}

export async function updateVisionDoc(
  docId: string, input: VisionUpdateInput
) {
  const db = getDb()
  const [result] = await db.update(visionDocument)
    .set({
      ...(input.title && { title: input.title }),
      ...(input.content && { content: input.content }),
      ...(input.category && { category: input.category }),
    })
    .where(and(
      eq(visionDocument.id, docId),
      eq(visionDocument.isActive, true),
    ))
    .returning()
  return result ?? null
}

export async function deleteVisionDoc(docId: string) {
  const db = getDb()
  await db.update(visionDocument)
    .set({ isActive: false })
    .where(eq(visionDocument.id, docId))
}

export async function getActiveVisionDocs(orgId: string, category?: string) {
  const db = getDb()
  const conditions = [
    eq(visionDocument.orgId, orgId),
    eq(visionDocument.isActive, true),
  ]
  if (category) {
    conditions.push(eq(visionDocument.category, category as typeof visionDocument.category.enumValues[number]))
  }
  return db.select().from(visionDocument)
    .where(and(...conditions))
    .orderBy(desc(visionDocument.createdAt))
}

export async function getVisionDoc(docId: string) {
  const db = getDb()
  const [result] = await db.select().from(visionDocument)
    .where(and(
      eq(visionDocument.id, docId),
      eq(visionDocument.isActive, true),
    ))
    .limit(1)
  return result ?? null
}
```

### `packages/data-ops/src/queries/activity.ts`

```ts
import { getDb } from "../database/setup"
import { activityLog } from "../drizzle/schema"
import { eq, and, desc, lt } from "drizzle-orm"

export async function getActivities(
  orgId: string,
  limit: number,
  cursor?: string
) {
  const db = getDb()
  const conditions = [eq(activityLog.orgId, orgId)]

  if (cursor) {
    conditions.push(lt(activityLog.id, cursor)) // ulid is lexicographically sorted
  }

  return db.select().from(activityLog)
    .where(and(...conditions))
    .orderBy(desc(activityLog.createdAt))
    .limit(limit)
}

export async function getActivity(activityId: string) {
  const db = getDb()
  const [result] = await db.select().from(activityLog)
    .where(eq(activityLog.id, activityId))
    .limit(1)
  return result ?? null
}
```

Cursor-based pagination via ULID comparison -- no offset needed. Client sends last activity ID as cursor, query returns next batch.

---

## Service Layer

### `apps/data-service/src/hono/services/vision-service.ts`

Thin wrapper. Admin-only checks for create/update/delete.

```ts
interface VisionServiceDeps {
  userId: string
  orgId: string
  memberRole: "admin" | "member"
}

export function createVisionService(deps: VisionServiceDeps) {
  return {
    async list(category?: string) {
      return getActiveVisionDocs(deps.orgId, category)
    },
    async get(docId: string) {
      return getVisionDoc(docId)
    },
    async create(input: VisionCreateInput) {
      if (deps.memberRole !== "admin") throw new ForbiddenError()
      return createVisionDoc(deps.orgId, input, deps.userId)
    },
    async update(docId: string, input: VisionUpdateInput) {
      if (deps.memberRole !== "admin") throw new ForbiddenError()
      return updateVisionDoc(docId, input)
    },
    async remove(docId: string) {
      if (deps.memberRole !== "admin") throw new ForbiddenError()
      return deleteVisionDoc(docId)
    },
  }
}
```

### `apps/data-service/src/hono/services/activity-service.ts`

```ts
export function createActivityService(orgId: string) {
  return {
    async feed(limit: number, cursor?: string) {
      return getActivities(orgId, limit, cursor)
    },
    async get(activityId: string) {
      return getActivity(activityId)
    },
  }
}
```

---

## Handlers

### `apps/data-service/src/hono/handlers/vision-handlers.ts`

```ts
import { Hono } from "hono"

const vision = new Hono()

vision.post("/", async (c) => {
  const body = VisionCreateRequestSchema.parse(await c.req.json())
  const svc = createVisionService(extractDeps(c))
  const doc = await svc.create(body)
  return c.json(doc, 201)
})

vision.get("/", async (c) => {
  const { category } = VisionListQuerySchema.parse(c.req.query())
  const svc = createVisionService(extractDeps(c))
  return c.json(await svc.list(category))
})

vision.get("/:docId", async (c) => {
  const svc = createVisionService(extractDeps(c))
  const doc = await svc.get(c.req.param("docId"))
  if (!doc) return c.json({ error: "Not found" }, 404)
  return c.json(doc)
})

vision.patch("/:docId", async (c) => {
  const body = VisionUpdateRequestSchema.parse(await c.req.json())
  const svc = createVisionService(extractDeps(c))
  const doc = await svc.update(c.req.param("docId"), body)
  if (!doc) return c.json({ error: "Not found" }, 404)
  return c.json(doc)
})

vision.delete("/:docId", async (c) => {
  const svc = createVisionService(extractDeps(c))
  await svc.remove(c.req.param("docId"))
  return c.json({ ok: true })
})

export default vision
```

### `apps/data-service/src/hono/handlers/activity-handlers.ts`

```ts
const activities = new Hono()

activities.get("/", async (c) => {
  const { limit, cursor } = ActivityFeedQuerySchema.parse(c.req.query())
  const svc = createActivityService(c.var.orgMember.orgId)
  const data = await svc.feed(limit, cursor)
  return c.json({
    data,
    nextCursor: data.length === limit ? data[data.length - 1].id : null,
  })
})

activities.get("/:id", async (c) => {
  const svc = createActivityService(c.var.orgMember.orgId)
  const activity = await svc.get(c.req.param("id"))
  if (!activity) return c.json({ error: "Not found" }, 404)
  return c.json(activity)
})

export default activities
```

---

## Frontend Pages

All pages use `createServerFn()` to call data-service via service binding. Use existing UI primitives (shadcn-style components from project).

### Vision Page (`vision.tsx`)

**Layout:**
- Top: category filter (4 buttons: rules/decisions/context/goals + "all") for the doc list
- Doc list: simple list showing title, category badge, last edited date
- Click doc -> navigates to inline editor below (or separate route `vision/$docId`)
- "New Document" button -> empty editor

**Editor (inline or sub-route):**
- Title input (text field)
- Category select (dropdown: rules/decisions/context/goals)
- Content textarea (full width, tall, monospace font -- writing markdown)
- Save button (POST or PATCH)
- Delete button (admin only, with confirm dialog)
- No live preview. No markdown rendering. Just edit raw text.

**Server functions:**

```ts
// apps/user-application/src/server/vision.ts
import { createServerFn } from "@tanstack/start"

export const getVisionDocs = createServerFn({ method: "GET" })
  .validator(z.object({ orgId: z.string(), category: z.string().optional() }))
  .handler(async ({ data }) => {
    // call data-service via service binding
  })

export const getVisionDoc = createServerFn({ method: "GET" })
  .validator(z.object({ orgId: z.string(), docId: z.string() }))
  .handler(async ({ data }) => { /* ... */ })

export const createVisionDoc = createServerFn({ method: "POST" })
  .validator(VisionCreateRequestSchema.extend({ orgId: z.string() }))
  .handler(async ({ data }) => { /* ... */ })

export const updateVisionDoc = createServerFn({ method: "POST" })
  .validator(VisionUpdateRequestSchema.extend({ orgId: z.string(), docId: z.string() }))
  .handler(async ({ data }) => { /* ... */ })

export const deleteVisionDoc = createServerFn({ method: "POST" })
  .validator(z.object({ orgId: z.string(), docId: z.string() }))
  .handler(async ({ data }) => { /* ... */ })
```

### Activity Feed Page (`activities.tsx`)

**Layout:**
- Simple reverse-chronological list, newest first
- Each entry: user name/email, summary text, timestamp (relative: "2h ago")
- Collapsible sections per entry: files changed, decisions made (only if non-empty)
- Tags shown as small badges
- "Load more" button at bottom (sends last item's ID as cursor, appends results)
- Default: show last 50

No filters. No date picker. No user dropdown. Just the raw feed.

**Server functions:**

```ts
// apps/user-application/src/server/activity.ts
export const getActivityFeed = createServerFn({ method: "GET" })
  .validator(z.object({
    orgId: z.string(),
    limit: z.number().default(50),
    cursor: z.string().optional(),
  }))
  .handler(async ({ data }) => { /* ... */ })
```

---

## Auth & Permissions

| Action | Who |
|--------|-----|
| Create/edit/delete vision docs | admin only |
| View vision docs | admin + member |
| View activity feed | admin + member |

Role check in service layer. `orgMemberMiddleware` (from Phase 1) sets role on context.

---

## Implementation Order

1. Zod schemas: `vision.ts`, `activity.ts`
2. Queries: `vision.ts`, `activity.ts`
3. Build data-ops
4. Services: `vision-service.ts`, `activity-service.ts`
5. Handlers: `vision-handlers.ts`, `activity-handlers.ts`
6. Register routes in `app.ts`
7. Server functions in user-application
8. Vision page (list + editor)
9. Activity feed page
10. Manual test with seed data + MCP-generated activities

---

## Unresolved Questions

- Vision PATCH: full content replace or partial merge with existing? (leaning full replace since textarea sends entire content)
- `version` column still in schema from Phase 1 -- always set to 1? Or drop column in migration?
- Category filter on vision list: query param or client-side filter? (few docs per org, client-side might suffice)
- Activity feed: show user's display name or email? Need join with auth_user table or denormalize name into activity_log
- Load-more vs infinite scroll? Load-more button simpler, but infinite scroll better UX
- Vision editor route: inline on vision.tsx or separate `vision/$docId.tsx` sub-route?
- Should activity entries link to anything? (no detail page designed yet beyond the expand)
- Max content size for vision doc textarea? (10KB? 50KB? need limit)

---