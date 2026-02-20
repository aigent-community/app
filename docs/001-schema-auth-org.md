# Phase 1 — Schema + Auth + Org

## Overview

Foundation layer: all Drizzle tables, Better Auth session auth, org CRUD, member management, API token (`at_`) CRUD. No vision/activity/memory/MCP — those are Phase 2+.

## Scope

**In scope:** DB schema (all tables, all phases), auth (email/password + session), orgs (create/read/update), members (invite/roles/remove), API tokens (generate/revoke/list), invite flow for unregistered users, rate limiting baseline.

**Out of scope:** vision docs, activity log, shared memory, MCP server, dashboard UI, Google OAuth.

---

## 1. Drizzle Schema

All tables created now (even Phase 2+ tables) so migration is single atomic step.

### 1.1 Enums

```
packages/data-ops/src/drizzle/schema.ts
```

```ts
import { pgEnum } from "drizzle-orm/pg-core"

export const orgMemberRoleEnum = pgEnum("org_member_role", ["admin", "member"])
export const visionCategoryEnum = pgEnum("vision_category", ["rules", "decisions", "context", "goals"])
```

No `memoryCategoryEnum` — premature. Categories will emerge from real usage.

### 1.2 Tables

All in `packages/data-ops/src/drizzle/schema.ts`. IDs are text (ULID), not UUID. FK references point to `auth_user` from `auth-schema.ts`.

#### organization

```ts
export const organization = pgTable("organization", {
  id: text("id").primaryKey(),                    // ulid
  name: text("name").notNull(),
  slug: text("slug").notNull().unique(),
  ownerId: text("owner_id").notNull().references(() => auth_user.id, { onDelete: "cascade" }),
  maxSeats: integer("max_seats").notNull().default(5),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow().notNull(),
})
```

#### organization_member

```ts
export const organizationMember = pgTable("organization_member", {
  id: text("id").primaryKey(),                    // ulid
  orgId: text("org_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  userId: text("user_id").references(() => auth_user.id, { onDelete: "cascade" }),
  email: text("email").notNull(),                 // always stored; userId null until user signs up
  role: orgMemberRoleEnum("role").notNull().default("member"),
  joinedAt: timestamp("joined_at").defaultNow().notNull(),
  isActive: boolean("is_active").notNull().default(true),
})
```

`userId` is nullable — pending invites have `email` but no `userId` until the invited user registers and claims the membership.

#### vision_document

```ts
export const visionDocument = pgTable("vision_document", {
  id: text("id").primaryKey(),                    // ulid
  orgId: text("org_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  title: text("title").notNull(),
  content: text("content").notNull(),
  category: visionCategoryEnum("category").notNull(),
  isActive: boolean("is_active").notNull().default(true),
  createdBy: text("created_by").notNull().references(() => auth_user.id),
  createdAt: timestamp("created_at").defaultNow().notNull(),
})
```

No `version` column — single active doc per title per org. Versioning added later if needed. Edits update in-place.

#### activity_log

```ts
export const activityLog = pgTable("activity_log", {
  id: text("id").primaryKey(),                    // ulid
  orgId: text("org_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  userId: text("user_id").notNull().references(() => auth_user.id),
  summary: text("summary").notNull(),
  filesChanged: jsonb("files_changed").$type<string[]>(),
  decisionsMade: jsonb("decisions_made").$type<string[]>(),
  tags: jsonb("tags").$type<string[]>(),
  sessionId: text("session_id"),
  createdAt: timestamp("created_at").defaultNow().notNull(),
})
```

#### shared_memory

```ts
export const sharedMemory = pgTable("shared_memory", {
  id: text("id").primaryKey(),                    // ulid
  orgId: text("org_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  content: text("content").notNull(),
  category: text("category"),                     // plain text, freeform — no enum
  sourceActivityId: text("source_activity_id").references(() => activityLog.id),
  createdBy: text("created_by").notNull().references(() => auth_user.id),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow().notNull(),
})
```

`category` is nullable plain text — no enum. Let categories emerge organically.

#### api_token

```ts
export const apiToken = pgTable("api_token", {
  id: text("id").primaryKey(),                    // ulid
  userId: text("user_id").notNull().references(() => auth_user.id, { onDelete: "cascade" }),
  orgId: text("org_id").notNull().references(() => organization.id, { onDelete: "cascade" }),
  tokenHash: text("token_hash").notNull(),        // SHA-256 of at_xxx
  name: text("name").notNull(),
  lastUsedAt: timestamp("last_used_at"),
  expiresAt: timestamp("expires_at"),
  isActive: boolean("is_active").notNull().default(true),
  createdAt: timestamp("created_at").defaultNow().notNull(),
})
```

### 1.3 Relations

```
packages/data-ops/src/drizzle/relations.ts
```

```ts
import { relations } from "drizzle-orm/relations"
import { auth_user } from "./auth-schema"
import {
  organization, organizationMember, visionDocument,
  activityLog, sharedMemory, apiToken
} from "./schema"

export const organizationRelations = relations(organization, ({ one, many }) => ({
  owner: one(auth_user, { fields: [organization.ownerId], references: [auth_user.id] }),
  members: many(organizationMember),
  visionDocuments: many(visionDocument),
  activityLogs: many(activityLog),
  sharedMemories: many(sharedMemory),
  apiTokens: many(apiToken),
}))

export const organizationMemberRelations = relations(organizationMember, ({ one }) => ({
  organization: one(organization, { fields: [organizationMember.orgId], references: [organization.id] }),
  user: one(auth_user, { fields: [organizationMember.userId], references: [auth_user.id] }),
}))

export const apiTokenRelations = relations(apiToken, ({ one }) => ({
  user: one(auth_user, { fields: [apiToken.userId], references: [auth_user.id] }),
  organization: one(organization, { fields: [apiToken.orgId], references: [organization.id] }),
}))

export const visionDocumentRelations = relations(visionDocument, ({ one }) => ({
  organization: one(organization, { fields: [visionDocument.orgId], references: [organization.id] }),
  creator: one(auth_user, { fields: [visionDocument.createdBy], references: [auth_user.id] }),
}))

export const activityLogRelations = relations(activityLog, ({ one }) => ({
  organization: one(organization, { fields: [activityLog.orgId], references: [organization.id] }),
  user: one(auth_user, { fields: [activityLog.userId], references: [auth_user.id] }),
}))

export const sharedMemoryRelations = relations(sharedMemory, ({ one }) => ({
  organization: one(organization, { fields: [sharedMemory.orgId], references: [organization.id] }),
  creator: one(auth_user, { fields: [sharedMemory.createdBy], references: [auth_user.id] }),
  sourceActivity: one(activityLog, { fields: [sharedMemory.sourceActivityId], references: [activityLog.id] }),
}))
```

### 1.4 Migration

```bash
pnpm --filter @repo/data-ops drizzle:dev:generate
pnpm --filter @repo/data-ops drizzle:dev:migrate
pnpm --filter @repo/data-ops build
```

Remove boilerplate `clients` table from `schema.ts` in same migration.

---

## 2. Better Auth

Auth already set up in boilerplate. Phase 1 changes:

### 2.1 Keep Existing

- `packages/data-ops/src/auth/setup.ts` — `createBetterAuth()` with email/password
- `packages/data-ops/src/auth/server.ts` — `setAuth()` / `getAuth()` singleton
- `packages/data-ops/src/drizzle/auth-schema.ts` — `auth_user`, `auth_session`, `auth_account`, `auth_verification`

### 2.2 New Auth Middleware (session-based)

Replace the existing bearer-token `authMiddleware` with Better Auth session middleware for `/api/*` routes.

```
apps/data-service/src/hono/middleware/auth.ts
```

```ts
import { getAuth } from "@repo/data-ops/auth/server"
import { createMiddleware } from "hono/factory"

interface AuthUser {
  id: string
  name: string
  email: string
}

interface AuthSession {
  id: string
  userId: string
  expiresAt: Date
}

interface AuthEnv {
  Bindings: Env
  Variables: {
    user: AuthUser
    session: AuthSession
  }
}

export const sessionAuth = createMiddleware<AuthEnv>(async (c, next) => {
  const auth = getAuth()
  const session = await auth.api.getSession({
    headers: c.req.raw.headers,
  })
  if (!session) {
    return c.json({ error: "Unauthorized" }, 401)
  }
  c.set("user", session.user)
  c.set("session", session.session)
  await next()
})
```

### 2.3 Auth Routes

Expose Better Auth handler at `/api/auth/*` in Hono app.

```
apps/data-service/src/hono/app.ts
```

```ts
import { getAuth } from "@repo/data-ops/auth/server"

App.all("/api/auth/*", (c) => {
  const auth = getAuth()
  return auth.handler(c.req.raw)
})
```

---

## 3. Zod Schemas

### 3.1 Organization

```
packages/data-ops/src/zod-schema/organization.ts
```

```ts
import { z } from "zod"

const slugRegex = /^[a-z0-9-]+$/

export const OrganizationSchema = z.object({
  id: z.string(),
  name: z.string(),
  slug: z.string(),
  ownerId: z.string(),
  maxSeats: z.number(),
  isActive: z.boolean(),
  createdAt: z.date(),
})

export const OrgCreateRequestSchema = z.object({
  name: z.string().min(1).max(100),
  slug: z.string().min(2).max(50).regex(slugRegex, "lowercase alphanumeric + hyphens only"),
})

export const OrgUpdateRequestSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  slug: z.string().min(2).max(50).regex(slugRegex).optional(),
}).refine(d => d.name || d.slug, { message: "At least one field required" })

export const OrgIdParamSchema = z.object({
  id: z.string(),
})

export type Organization = z.infer<typeof OrganizationSchema>
export type OrgCreateInput = z.infer<typeof OrgCreateRequestSchema>
export type OrgUpdateInput = z.infer<typeof OrgUpdateRequestSchema>
```

### 3.2 Member

```
packages/data-ops/src/zod-schema/member.ts
```

```ts
import { z } from "zod"

export const orgMemberRoleSchema = z.enum(["admin", "member"])

export const MemberSchema = z.object({
  id: z.string(),
  orgId: z.string(),
  userId: z.string().nullable(),
  email: z.string().email(),
  role: orgMemberRoleSchema,
  joinedAt: z.date(),
  isActive: z.boolean(),
})

export const MemberInviteRequestSchema = z.object({
  email: z.string().email(),
  role: orgMemberRoleSchema.default("member"),
})

export const MemberUpdateRequestSchema = z.object({
  role: orgMemberRoleSchema.optional(),
  isActive: z.boolean().optional(),
}).refine(d => d.role !== undefined || d.isActive !== undefined, {
  message: "At least one field required",
})

export const MemberIdParamSchema = z.object({
  id: z.string(),
  memberId: z.string(),
})

export type Member = z.infer<typeof MemberSchema>
export type MemberInviteInput = z.infer<typeof MemberInviteRequestSchema>
export type MemberUpdateInput = z.infer<typeof MemberUpdateRequestSchema>
```

### 3.3 API Token

```
packages/data-ops/src/zod-schema/api-token.ts
```

```ts
import { z } from "zod"

export const ApiTokenSchema = z.object({
  id: z.string(),
  userId: z.string(),
  orgId: z.string(),
  name: z.string(),
  lastUsedAt: z.date().nullable(),
  expiresAt: z.date().nullable(),
  isActive: z.boolean(),
  createdAt: z.date(),
})

export const TokenCreateRequestSchema = z.object({
  orgId: z.string(),
  name: z.string().min(1).max(100),
  expiresAt: z.coerce.date().optional(),
})

export const TokenIdParamSchema = z.object({
  id: z.string(),
})

export const TokenCreateResponseSchema = ApiTokenSchema.extend({
  token: z.string(),  // at_xxx — shown once
})

export type ApiToken = z.infer<typeof ApiTokenSchema>
export type TokenCreateInput = z.infer<typeof TokenCreateRequestSchema>
export type TokenCreateResponse = z.infer<typeof TokenCreateResponseSchema>
```

---

## 4. Queries

### 4.1 Organization Queries

```
packages/data-ops/src/queries/organization.ts
```

Functions:
- `createOrganization(data: OrgCreateInput, ownerId: string): Promise<Organization>` — insert org + insert owner as admin member (transaction)
- `getOrganization(orgId: string): Promise<Organization | null>`
- `getOrganizationBySlug(slug: string): Promise<Organization | null>`
- `getUserOrganizations(userId: string): Promise<Organization[]>` — join via `organization_member`
- `updateOrganization(orgId: string, data: OrgUpdateInput): Promise<Organization | null>`

Key: `createOrganization` uses Drizzle transaction — inserts org row then inserts the creator as `role: "admin"` in `organization_member` (with `email` populated from session user).

### 4.2 Member Queries

```
packages/data-ops/src/queries/member.ts
```

Functions:
- `getOrgMembers(orgId: string): Promise<MemberWithUser[]>` — left join with `auth_user` for name/email (pending invites have null user)
- `getMember(memberId: string): Promise<Member | null>`
- `getMemberByUserAndOrg(userId: string, orgId: string): Promise<Member | null>`
- `getMemberByEmailAndOrg(email: string, orgId: string): Promise<Member | null>`
- `addMember(orgId: string, email: string, role: "admin" | "member", userId?: string): Promise<Member>`
- `claimMembership(email: string, userId: string): Promise<number>` — sets `userId` on all pending rows matching email
- `updateMember(memberId: string, data: MemberUpdateInput): Promise<Member | null>`
- `removeMember(memberId: string): Promise<boolean>` — sets `is_active = false`

#### Invite Flow (resolved)

1. Admin invites by email via `POST /api/orgs/:id/members/invite`
2. Handler calls `getMemberByEmailAndOrg` — if already exists, return error "already invited/member"
3. Check seat limit (`max_seats`) — hard block if at capacity
4. Insert `organization_member` row with `email`, `userId = null`, `role`
5. If invited user already has an account (`auth_user` lookup by email), set `userId` immediately
6. If not registered: row stays with `userId = null` (pending invite)
7. On signup: call `claimMembership(email, newUserId)` to backfill `userId` on all pending rows
8. Phase 2+: send invite email with signup link

### 4.3 API Token Queries

```
packages/data-ops/src/queries/api-token.ts
```

Functions:
- `createApiToken(data: { userId, orgId, name, expiresAt?, tokenHash }): Promise<ApiToken>`
- `getUserTokens(userId: string): Promise<ApiToken[]>`
- `getTokenByHash(tokenHash: string): Promise<ApiTokenWithOrg | null>` — for MCP auth (Phase 3)
- `revokeToken(tokenId: string, userId: string): Promise<boolean>` — sets `is_active = false`, verifies ownership
- `updateTokenLastUsed(tokenId: string): Promise<void>`

Token generation (in service layer, not query):
1. Generate 32 random bytes -> base64url -> prepend `at_`
2. SHA-256 hash the full token string
3. Store hash in DB, return plaintext to user **once**

---

## 5. API Endpoints (Hono)

All routes require `sessionAuth` middleware (authenticated user via Better Auth session).

### 5.1 Route Files

```
apps/data-service/src/hono/handlers/org-handlers.ts
apps/data-service/src/hono/handlers/member-handlers.ts
apps/data-service/src/hono/handlers/token-handlers.ts

apps/data-service/src/hono/services/org-service.ts
apps/data-service/src/hono/services/member-service.ts
apps/data-service/src/hono/services/token-service.ts
```

### 5.2 Organization Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/orgs` | session | Create org (caller becomes admin) |
| GET | `/api/orgs` | session | List user's orgs |
| GET | `/api/orgs/:id` | session + member | Get org details |
| PATCH | `/api/orgs/:id` | session + admin | Update org |

### 5.3 Member Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/orgs/:id/members/invite` | session + admin | Invite user by email |
| GET | `/api/orgs/:id/members` | session + member | List members (includes pending) |
| PATCH | `/api/orgs/:id/members/:memberId` | session + admin | Update role/status |
| DELETE | `/api/orgs/:id/members/:memberId` | session + admin | Deactivate member |

### 5.4 Token Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/tokens` | session | Generate new `at_` token |
| GET | `/api/tokens` | session | List user's tokens |
| DELETE | `/api/tokens/:id` | session | Revoke token |

### 5.5 Auth Routes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| ALL | `/api/auth/*` | none | Better Auth handler (signup, login, session, etc.) |

### 5.6 App Registration

```
apps/data-service/src/hono/app.ts
```

```ts
App.all("/api/auth/*", (c) => getAuth().handler(c.req.raw))
App.use("/api/orgs/*", sessionAuth)
App.use("/api/tokens/*", sessionAuth)
App.route("/api/orgs", orgs)
App.route("/api/tokens", tokens)
```

---

## 6. Authorization Middleware

Two levels beyond session auth:

### 6.1 Org Membership Check

```
apps/data-service/src/hono/middleware/org-member.ts
```

Middleware that checks if `c.var.user.id` is active member of `:id` org. Sets `c.var.member` with role info. Used on all `/api/orgs/:id/*` routes.

### 6.2 Admin-Only Check

```
apps/data-service/src/hono/middleware/org-admin.ts
```

Middleware that checks `c.var.member.role === "admin"`. Used on mutation routes (PATCH org, invite, manage members).

---

## 7. Token Generation

```
apps/data-service/src/hono/services/token-service.ts
```

```ts
import { ulid } from "ulid"

function generateToken(): { plaintext: string; hash: string } {
  const bytes = crypto.getRandomValues(new Uint8Array(32))
  const encoded = btoa(String.fromCharCode(...bytes))
    .replace(/\+/g, "-").replace(/\//g, "_").replace(/=/g, "")
  const plaintext = `at_${encoded}`
  const hashBuffer = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(plaintext))
  const hash = Array.from(new Uint8Array(hashBuffer)).map(b => b.toString(16).padStart(2, "0")).join("")
  return { plaintext, hash }
}
```

Token prefix `at_` per PLAN.md resolved decision #9. Plaintext returned **once** on creation, never stored. DB stores only SHA-256 hash.

---

## 8. ULID Generation

All entity IDs use ULID (text PK). Use `ulid` package. Generate in service layer before inserting.

```ts
import { ulid } from "ulid"
const id = ulid()
```

---

## 9. Rate Limiting

MCP endpoint is public from Phase 2. Baseline rate limiting needed from day 1.

### 9.1 Phase 1 Scope

Add IP-based rate limiting middleware to the Hono app. Applied globally on `/api/*` routes.

```
apps/data-service/src/hono/middleware/rate-limit.ts
```

Implementation: in-memory sliding window per IP. CF Workers have no shared state across isolates, so this limits per-isolate only — good enough for MVP. Phase 5 upgrades to Durable Objects or KV for global rate limiting.

Limits:
- `/api/auth/*`: 10 req/min per IP (brute-force protection)
- `/api/*` (authenticated): 60 req/min per IP
- `/mcp/*` (Phase 2+): 30 req/min per token

### 9.2 Middleware Shape

```ts
interface RateLimitConfig {
  windowMs: number
  maxRequests: number
  keyFn: (c: Context) => string  // IP, token, userId
}

export function rateLimiter(config: RateLimitConfig): MiddlewareHandler
```

Returns 429 with `Retry-After` header on limit exceeded.

---

## 10. Membership Claim on Signup

When a new user registers via Better Auth, backfill any pending invites.

### 10.1 Better Auth Hook

```
packages/data-ops/src/auth/setup.ts
```

Add `onUserCreated` hook (or equivalent Better Auth event) that calls `claimMembership(email, userId)` to set `userId` on all `organization_member` rows where `email` matches and `userId` is null.

This ensures invited-but-unregistered users automatically join their orgs on signup.

---

## 11. File Map

### packages/data-ops

| File | Action |
|------|--------|
| `src/drizzle/schema.ts` | Replace `clients` table with all 6 tables + 2 enums |
| `src/drizzle/relations.ts` | Uncomment, add all relations |
| `src/auth/setup.ts` | Add onUserCreated hook for membership claim |
| `src/queries/organization.ts` | New — org CRUD queries |
| `src/queries/member.ts` | New — member queries + claimMembership |
| `src/queries/api-token.ts` | New — token queries |
| `src/zod-schema/organization.ts` | New — org schemas |
| `src/zod-schema/member.ts` | New — member schemas |
| `src/zod-schema/api-token.ts` | New — token schemas |
| `src/queries/client.ts` | Delete (boilerplate) |
| `src/zod-schema/client.ts` | Delete (boilerplate) |

### apps/data-service

| File | Action |
|------|--------|
| `src/hono/app.ts` | Add auth routes, org routes, token routes, sessionAuth middleware |
| `src/hono/middleware/auth.ts` | Replace bearer auth with Better Auth session middleware |
| `src/hono/middleware/org-member.ts` | New — org membership check |
| `src/hono/middleware/org-admin.ts` | New — admin role check |
| `src/hono/middleware/rate-limit.ts` | New — IP-based rate limiting |
| `src/hono/handlers/org-handlers.ts` | New — org CRUD handlers |
| `src/hono/handlers/member-handlers.ts` | New — member handlers |
| `src/hono/handlers/token-handlers.ts` | New — token handlers |
| `src/hono/services/org-service.ts` | New — org business logic |
| `src/hono/services/member-service.ts` | New — member + invite logic |
| `src/hono/services/token-service.ts` | New — token generation + revocation |
| `src/hono/handlers/client-handlers.ts` | Delete (boilerplate) |
| `src/hono/services/client-service.ts` | Delete (boilerplate) |

---

## 12. Implementation Order

1. Schema: 2 enums + all 6 tables in `schema.ts`, relations in `relations.ts`
2. Generate + run migration
3. Zod schemas: organization, member, api-token
4. Queries: organization, member (incl. claimMembership), api-token
5. Auth: `sessionAuth` middleware, onUserCreated hook for membership claim
6. Authorization: `orgMember`, `orgAdmin` middleware
7. Rate limiting: `rateLimiter` middleware
8. Services: org-service, member-service, token-service
9. Handlers: org-handlers, member-handlers, token-handlers
10. Wire routes in `app.ts`
11. Delete boilerplate (clients)
12. Build + test

---

## Unresolved Questions

- Better Auth `onUserCreated` hook — exact API? Need to verify Better Auth supports post-signup hooks or if we use a Hono middleware on the signup response instead
- `max_seats` — count pending invites toward limit or only claimed members?
- Org deletion: soft-delete (`is_active = false`) or hard delete with cascade?
- `GET /api/orgs` — return only active orgs or include deactivated for admins?
- Token expiry: auto-filter expired tokens in queries, or also run a scheduled cleanup?
- Org slug: auto-generate from name, or always user-provided?
- Should org owner transfer be supported in Phase 1?
- Rate limit per-isolate is leaky — acceptable for MVP or need KV-backed from day 1?

---