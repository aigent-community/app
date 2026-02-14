# 001 - Phase 1: Schema + Auth (Foundation)

## Overview

Drizzle schema, Zod validation, Hono CRUD routes, and credit initialization for the TokSwap marketplace foundation. Covers all six new tables, provider config + token pool management APIs, and automatic credit balance creation on signup.

## Context & Background

TokSwap is an LLM token sharing marketplace. Before sessions, agents, or marketplace features can exist, the data layer must be in place. This phase establishes the complete database schema for all domain entities and implements the first two CRUD surfaces (provider config, token pools) plus the credit system bootstrap.

## Goals

- All six new tables with proper constraints, indexes, FKs
- Provider config CRUD (create/update/get)
- Token pool CRUD (create/list/update/delete/pause/resume)
- Credit balance auto-creation on signup (500 bonus)
- Zod validation on all API boundaries
- Auth-gated endpoints (Better Auth session)

## Non-Goals

- Session lifecycle (Phase 2)
- Marketplace browsing (Phase 3)
- Credit reserve/settle engine (Phase 2)
- WebSocket/Durable Objects (Phase 2)
- CLI agent (Phase 4)
- API key encryption (Phase 5)

---

## 1. Database Schema

All tables go in `packages/data-ops/src/drizzle/schema.ts`. Keep existing `clients` table. Relations go in `relations.ts`.

### 1.1 Enums

```ts
// packages/data-ops/src/drizzle/schema.ts
import {
  pgTable,
  pgEnum,
  text,
  uuid,
  boolean,
  integer,
  bigint,
  real,
  timestamp,
  jsonb,
  index,
  uniqueIndex,
} from "drizzle-orm/pg-core";
import { auth_user } from "./auth-schema";

export const providerModeEnum = pgEnum("provider_mode", [
  "local_proxy",
  "api_key",
]);

export const poolStatusEnum = pgEnum("pool_status", [
  "active",
  "paused",
  "depleted",
  "revoked",
]);

export const sessionStatusEnum = pgEnum("session_status", [
  "pending",
  "active",
  "ending",
  "completed",
  "aborted",
  "timeout",
]);

export const sessionEndReasonEnum = pgEnum("session_end_reason", [
  "tokens_exhausted",
  "time_window",
  "user_ended",
  "provider_revoked",
  "provider_disconnected",
  "error",
]);

export const creditTypeEnum = pgEnum("credit_type", [
  "signup_bonus",
  "provider_earning",
  "consumer_spend",
  "refund",
]);
```

### 1.2 provider_config

```ts
export const providerConfig = pgTable(
  "provider_config",
  {
    id: text("id").primaryKey(), // ulid
    userId: text("user_id")
      .notNull()
      .references(() => auth_user.id, { onDelete: "cascade" }),
    mode: providerModeEnum("mode").notNull(),
    llmProvider: text("llm_provider").notNull().default("claude"),
    encryptedApiKey: text("encrypted_api_key"),
    modelAllowlist: jsonb("model_allowlist").$type<string[]>().default([]),
    isActive: boolean("is_active").notNull().default(true),
    lastSeenAt: timestamp("last_seen_at"),
    reputationScore: real("reputation_score").notNull().default(5.0),
    totalSessionsServed: integer("total_sessions_served").notNull().default(0),
    totalTokensServed: bigint("total_tokens_served", { mode: "number" })
      .notNull()
      .default(0),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at")
      .defaultNow()
      .$onUpdate(() => new Date())
      .notNull(),
  },
  (table) => [
    uniqueIndex("provider_config_user_id_idx").on(table.userId),
  ]
);
```

One provider_config per user (enforced by unique index on `userId`).

### 1.3 token_pool

```ts
export const tokenPool = pgTable(
  "token_pool",
  {
    id: text("id").primaryKey(), // ulid
    providerId: text("provider_id")
      .notNull()
      .references(() => providerConfig.id, { onDelete: "cascade" }),
    name: text("name").notNull(),
    description: text("description"),
    status: poolStatusEnum("status").notNull().default("active"),
    availableFrom: text("available_from"), // "09:00" UTC
    availableTo: text("available_to"), // "17:00" UTC
    availableDays: jsonb("available_days").$type<number[]>().default([1, 2, 3, 4, 5]), // 0=Sun JS convention, Mon-Fri
    maxTokensPerSession: integer("max_tokens_per_session").notNull(),
    maxTokensPerDay: integer("max_tokens_per_day").notNull(),
    dailyTokensUsed: integer("daily_tokens_used").notNull().default(0),
    dailyTokensResetAt: timestamp("daily_tokens_reset_at").defaultNow().notNull(), // lazy reset: compare date on first request
    allowedUseCases: jsonb("allowed_use_cases").$type<string[]>().default([]),
    allowedModels: jsonb("allowed_models").$type<string[]>().notNull(),
    creditsPerInputKToken: integer("credits_per_input_k_token").notNull(),
    creditsPerOutputKToken: integer("credits_per_output_k_token").notNull(),
    systemPrompt: text("system_prompt"), // optional per-pool system prompt sent to Claude
    maxConcurrentSessions: integer("max_concurrent_sessions").notNull().default(1),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at")
      .defaultNow()
      .$onUpdate(() => new Date())
      .notNull(),
  },
  (table) => [
    index("token_pool_provider_id_idx").on(table.providerId),
    index("token_pool_status_idx").on(table.status),
  ]
);
```

### 1.4 llm_session

```ts
export const llmSession = pgTable(
  "llm_session",
  {
    id: text("id").primaryKey(), // ulid
    poolId: text("pool_id")
      .notNull()
      .references(() => tokenPool.id, { onDelete: "set null" }),
    consumerId: text("consumer_id")
      .notNull()
      .references(() => auth_user.id, { onDelete: "set null" }),
    providerId: text("provider_id")
      .notNull()
      .references(() => auth_user.id, { onDelete: "set null" }),
    model: text("model").notNull(), // claude model used for this session
    status: sessionStatusEnum("status").notNull().default("pending"),
    inputTokensUsed: integer("input_tokens_used").notNull().default(0),
    outputTokensUsed: integer("output_tokens_used").notNull().default(0),
    maxTokensBudget: integer("max_tokens_budget").notNull(), // output tokens only
    creditsCharged: integer("credits_charged").notNull().default(0),
    creditsReserved: integer("credits_reserved").notNull().default(0),
    endReason: sessionEndReasonEnum("end_reason"),
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at")
      .defaultNow()
      .$onUpdate(() => new Date())
      .notNull(),
  },
  (table) => [
    index("llm_session_pool_id_idx").on(table.poolId),
    index("llm_session_consumer_id_idx").on(table.consumerId),
    index("llm_session_provider_id_idx").on(table.providerId),
    index("llm_session_status_idx").on(table.status),
  ]
);
```

### 1.5 credit_balance

```ts
export const creditBalance = pgTable("credit_balance", {
  userId: text("user_id")
    .primaryKey()
    .references(() => auth_user.id, { onDelete: "cascade" }),
  available: integer("available").notNull().default(0),
  reserved: integer("reserved").notNull().default(0),
  updatedAt: timestamp("updated_at")
    .defaultNow()
    .$onUpdate(() => new Date())
    .notNull(),
});
```

### 1.6 credit_ledger

```ts
export const creditLedger = pgTable(
  "credit_ledger",
  {
    id: text("id").primaryKey(), // ulid
    userId: text("user_id")
      .notNull()
      .references(() => auth_user.id, { onDelete: "cascade" }),
    amount: integer("amount").notNull(), // +credit / -debit
    type: creditTypeEnum("type").notNull(),
    sessionId: text("session_id").references(() => llmSession.id),
    balanceAfter: integer("balance_after").notNull(),
    description: text("description"),
    createdAt: timestamp("created_at").defaultNow().notNull(),
  },
  (table) => [
    index("credit_ledger_user_id_idx").on(table.userId),
    index("credit_ledger_session_id_idx").on(table.sessionId),
    index("credit_ledger_created_at_idx").on(table.createdAt),
  ]
);
```

### 1.7 usage_log

```ts
export const usageLog = pgTable(
  "usage_log",
  {
    id: text("id").primaryKey(), // ulid
    sessionId: text("session_id")
      .notNull()
      .references(() => llmSession.id),
    requestIndex: integer("request_index").notNull(),
    inputTokens: integer("input_tokens").notNull(),
    outputTokens: integer("output_tokens").notNull(),
    model: text("model").notNull(),
    latencyMs: integer("latency_ms").notNull(),
    createdAt: timestamp("created_at").defaultNow().notNull(),
  },
  (table) => [
    index("usage_log_session_id_idx").on(table.sessionId),
  ]
);
```

### 1.8 Relations

```ts
// packages/data-ops/src/drizzle/relations.ts
import { relations } from "drizzle-orm/relations";
import { auth_user } from "./auth-schema";
import {
  providerConfig,
  tokenPool,
  llmSession,
  creditBalance,
  creditLedger,
  usageLog,
} from "./schema";

export const authUserRelations = relations(auth_user, ({ one, many }) => ({
  providerConfig: one(providerConfig, {
    fields: [auth_user.id],
    references: [providerConfig.userId],
  }),
  creditBalance: one(creditBalance, {
    fields: [auth_user.id],
    references: [creditBalance.userId],
  }),
  creditLedgerEntries: many(creditLedger),
  consumerSessions: many(llmSession, { relationName: "consumer" }),
  providerSessions: many(llmSession, { relationName: "provider" }),
}));

export const providerConfigRelations = relations(providerConfig, ({ one, many }) => ({
  user: one(auth_user, {
    fields: [providerConfig.userId],
    references: [auth_user.id],
  }),
  pools: many(tokenPool),
}));

export const tokenPoolRelations = relations(tokenPool, ({ one, many }) => ({
  provider: one(providerConfig, {
    fields: [tokenPool.providerId],
    references: [providerConfig.id],
  }),
  sessions: many(llmSession),
}));

export const llmSessionRelations = relations(llmSession, ({ one, many }) => ({
  pool: one(tokenPool, {
    fields: [llmSession.poolId],
    references: [tokenPool.id],
  }),
  consumer: one(auth_user, {
    fields: [llmSession.consumerId],
    references: [auth_user.id],
    relationName: "consumer",
  }),
  provider: one(auth_user, {
    fields: [llmSession.providerId],
    references: [auth_user.id],
    relationName: "provider",
  }),
  usageLogs: many(usageLog),
}));

export const creditBalanceRelations = relations(creditBalance, ({ one }) => ({
  user: one(auth_user, {
    fields: [creditBalance.userId],
    references: [auth_user.id],
  }),
}));

export const creditLedgerRelations = relations(creditLedger, ({ one }) => ({
  user: one(auth_user, {
    fields: [creditLedger.userId],
    references: [auth_user.id],
  }),
  session: one(llmSession, {
    fields: [creditLedger.sessionId],
    references: [llmSession.id],
  }),
}));

export const usageLogRelations = relations(usageLog, ({ one }) => ({
  session: one(llmSession, {
    fields: [usageLog.sessionId],
    references: [llmSession.id],
  }),
}));
```

### 1.9 Drizzle Config Update

Add new schema file to all three drizzle configs. The `tablesFilter: ["!auth_*"]` already excludes auth tables. No config change needed unless we split schema into multiple files. For Phase 1, keep everything in `schema.ts`.

---

## 2. Zod Schemas

### 2.1 Provider Config (`packages/data-ops/src/zod-schema/provider.ts`)

```ts
import { z } from "zod";

// ============================================
// Domain Models
// ============================================

export const ProviderModeSchema = z.enum(["local_proxy", "api_key"]);

export const ProviderConfigSchema = z.object({
  id: z.string(),
  userId: z.string(),
  mode: ProviderModeSchema,
  llmProvider: z.string(),
  encryptedApiKey: z.string().nullable(),
  modelAllowlist: z.array(z.string()),
  isActive: z.boolean(),
  lastSeenAt: z.string().nullable(),
  reputationScore: z.number(),
  totalSessionsServed: z.number(),
  totalTokensServed: z.number(),
  createdAt: z.string(),
  updatedAt: z.string(),
});

// ============================================
// Request Schemas
// ============================================

export const ProviderConfigCreateRequestSchema = z.object({
  mode: ProviderModeSchema,
  llmProvider: z.string().min(1).max(50).default("claude"),
  apiKey: z.string().min(1).max(500).optional(),
  modelAllowlist: z.array(z.string().min(1).max(100)).max(20).default([]),
}).refine(
  (data) => data.mode !== "api_key" || data.apiKey,
  { message: "apiKey required when mode is api_key", path: ["apiKey"] }
);

export const ProviderConfigUpdateRequestSchema = z.object({
  mode: ProviderModeSchema.optional(),
  llmProvider: z.string().min(1).max(50).optional(),
  apiKey: z.string().min(1).max(500).optional(),
  modelAllowlist: z.array(z.string().min(1).max(100)).max(20).optional(),
  isActive: z.boolean().optional(),
}).refine(
  (data) => Object.values(data).some((v) => v !== undefined),
  { message: "At least one field required" }
);

// ============================================
// Response Schemas
// ============================================

export const ProviderConfigResponseSchema = ProviderConfigSchema.omit({
  encryptedApiKey: true,
}).extend({
  hasApiKey: z.boolean(),
});

// ============================================
// Types
// ============================================

export type ProviderMode = z.infer<typeof ProviderModeSchema>;
export type ProviderConfig = z.infer<typeof ProviderConfigSchema>;
export type ProviderConfigCreateInput = z.infer<typeof ProviderConfigCreateRequestSchema>;
export type ProviderConfigUpdateInput = z.infer<typeof ProviderConfigUpdateRequestSchema>;
export type ProviderConfigResponse = z.infer<typeof ProviderConfigResponseSchema>;
```

### 2.2 Token Pool (`packages/data-ops/src/zod-schema/pool.ts`)

```ts
import { z } from "zod";

// ============================================
// Domain Models
// ============================================

export const PoolStatusSchema = z.enum(["active", "paused", "depleted", "revoked"]);

const timeRegex = /^([01]\d|2[0-3]):[0-5]\d$/;

export const TokenPoolSchema = z.object({
  id: z.string(),
  providerId: z.string(),
  name: z.string(),
  description: z.string().nullable(),
  status: PoolStatusSchema,
  availableFrom: z.string().nullable(),
  availableTo: z.string().nullable(),
  availableDays: z.array(z.number().int().min(0).max(6)),
  maxTokensPerSession: z.number().int().positive(),
  maxTokensPerDay: z.number().int().positive(),
  dailyTokensUsed: z.number().int(),
  allowedUseCases: z.array(z.string()),
  allowedModels: z.array(z.string()),
  creditsPerInputKToken: z.number().int().positive(),
  creditsPerOutputKToken: z.number().int().positive(),
  systemPrompt: z.string().nullable(),
  maxConcurrentSessions: z.number().int().positive(),
  createdAt: z.string(),
  updatedAt: z.string(),
});

// ============================================
// Request Schemas
// ============================================

export const PoolCreateRequestSchema = z.object({
  name: z.string().min(1, "Name required").max(100),
  description: z.string().max(500).optional(),
  availableFrom: z.string().regex(timeRegex, "Format: HH:MM").optional(),
  availableTo: z.string().regex(timeRegex, "Format: HH:MM").optional(),
  availableDays: z.array(z.number().int().min(0).max(6)).min(1).max(7).default([1, 2, 3, 4, 5]),
  maxTokensPerSession: z.number().int().min(1000).max(10_000_000),
  maxTokensPerDay: z.number().int().min(1000).max(100_000_000),
  allowedUseCases: z.array(z.string().min(1).max(50)).max(20).default([]),
  allowedModels: z.array(z.string().min(1).max(100)).min(1, "At least one model required").max(20),
  creditsPerInputKToken: z.number().int().min(1).max(10000),
  creditsPerOutputKToken: z.number().int().min(1).max(10000),
  systemPrompt: z.string().max(10_000).optional(),
  maxConcurrentSessions: z.number().int().min(1).max(100).default(1),
});

export const PoolUpdateRequestSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  description: z.string().max(500).nullable().optional(),
  availableFrom: z.string().regex(timeRegex).nullable().optional(),
  availableTo: z.string().regex(timeRegex).nullable().optional(),
  availableDays: z.array(z.number().int().min(0).max(6)).min(1).max(7).optional(),
  maxTokensPerSession: z.number().int().min(1000).max(10_000_000).optional(),
  maxTokensPerDay: z.number().int().min(1000).max(100_000_000).optional(),
  allowedUseCases: z.array(z.string().min(1).max(50)).max(20).optional(),
  allowedModels: z.array(z.string().min(1).max(100)).min(1).max(20).optional(),
  creditsPerInputKToken: z.number().int().min(1).max(10000).optional(),
  creditsPerOutputKToken: z.number().int().min(1).max(10000).optional(),
  systemPrompt: z.string().max(10_000).nullable().optional(),
  maxConcurrentSessions: z.number().int().min(1).max(100).optional(),
}).refine(
  (data) => Object.values(data).some((v) => v !== undefined),
  { message: "At least one field required" }
);

export const PoolIdParamSchema = z.object({
  id: z.string().min(1, "Pool ID required"),
});

// ============================================
// Response Schemas
// ============================================

export const TokenPoolResponseSchema = TokenPoolSchema;

export const TokenPoolListResponseSchema = z.object({
  data: z.array(TokenPoolSchema),
  pagination: z.object({
    total: z.number(),
    limit: z.number(),
    offset: z.number(),
    hasMore: z.boolean(),
  }),
});

// ============================================
// Types
// ============================================

export type PoolStatus = z.infer<typeof PoolStatusSchema>;
export type TokenPool = z.infer<typeof TokenPoolSchema>;
export type PoolCreateInput = z.infer<typeof PoolCreateRequestSchema>;
export type PoolUpdateInput = z.infer<typeof PoolUpdateRequestSchema>;
export type TokenPoolResponse = z.infer<typeof TokenPoolResponseSchema>;
export type TokenPoolListResponse = z.infer<typeof TokenPoolListResponseSchema>;
```

### 2.3 Credit (`packages/data-ops/src/zod-schema/credit.ts`)

```ts
import { z } from "zod";

// ============================================
// Domain Models
// ============================================

export const CreditTypeSchema = z.enum([
  "signup_bonus",
  "provider_earning",
  "consumer_spend",
  "refund",
]);

export const CreditBalanceSchema = z.object({
  userId: z.string(),
  available: z.number().int(),
  reserved: z.number().int(),
  updatedAt: z.string(),
});

export const CreditLedgerEntrySchema = z.object({
  id: z.string(),
  userId: z.string(),
  amount: z.number().int(),
  type: CreditTypeSchema,
  sessionId: z.string().nullable(),
  balanceAfter: z.number().int(),
  description: z.string().nullable(),
  createdAt: z.string(),
});

// ============================================
// Types
// ============================================

export type CreditType = z.infer<typeof CreditTypeSchema>;
export type CreditBalance = z.infer<typeof CreditBalanceSchema>;
export type CreditLedgerEntry = z.infer<typeof CreditLedgerEntrySchema>;
```

---

## 3. Queries

### 3.1 Provider Config (`packages/data-ops/src/queries/provider.ts`)

```ts
import { eq } from "drizzle-orm";
import { getDb } from "../database/setup";
import { providerConfig } from "../drizzle/schema";

interface ProviderConfigInsertData {
  id: string;
  userId: string;
  mode: "local_proxy" | "api_key";
  llmProvider: string;
  encryptedApiKey: string | null;
  modelAllowlist: string[];
}

export async function getProviderConfigByUserId(userId: string) {
  const db = getDb();
  const result = await db
    .select()
    .from(providerConfig)
    .where(eq(providerConfig.userId, userId));
  return result[0] ?? null;
}

export async function createProviderConfig(data: ProviderConfigInsertData) {
  const db = getDb();
  const [row] = await db.insert(providerConfig).values(data).returning();
  return row!;
}

export async function updateProviderConfig(
  id: string,
  data: Partial<{
    mode: "local_proxy" | "api_key";
    llmProvider: string;
    encryptedApiKey: string | null;
    modelAllowlist: string[];
    isActive: boolean;
  }>
) {
  const db = getDb();
  const result = await db
    .update(providerConfig)
    .set(data)
    .where(eq(providerConfig.id, id))
    .returning();
  return result[0] ?? null;
}
```

### 3.2 Token Pool (`packages/data-ops/src/queries/pool.ts`)

```ts
import { eq, count, and } from "drizzle-orm";
import { getDb } from "../database/setup";
import { tokenPool, providerConfig } from "../drizzle/schema";
import type { PaginationRequest } from "../zod-schema/client";

interface PoolInsertData {
  id: string;
  providerId: string;
  name: string;
  description?: string | null;
  availableFrom?: string | null;
  availableTo?: string | null;
  availableDays: number[];
  maxTokensPerSession: number;
  maxTokensPerDay: number;
  allowedUseCases: string[];
  allowedModels: string[];
  creditsPerInputKToken: number;
  creditsPerOutputKToken: number;
  maxConcurrentSessions: number;
}

export async function getPoolById(poolId: string) {
  const db = getDb();
  const result = await db.select().from(tokenPool).where(eq(tokenPool.id, poolId));
  return result[0] ?? null;
}

export async function getPoolsByProviderId(
  providerId: string,
  params: PaginationRequest
) {
  const db = getDb();
  const [data, countResult] = await Promise.all([
    db
      .select()
      .from(tokenPool)
      .where(eq(tokenPool.providerId, providerId))
      .limit(params.limit)
      .offset(params.offset),
    db
      .select({ total: count() })
      .from(tokenPool)
      .where(eq(tokenPool.providerId, providerId)),
  ]);
  const total = countResult[0]?.total ?? 0;
  return {
    data,
    pagination: {
      total,
      limit: params.limit,
      offset: params.offset,
      hasMore: params.offset + data.length < total,
    },
  };
}

export async function createPool(data: PoolInsertData) {
  const db = getDb();
  const [row] = await db.insert(tokenPool).values(data).returning();
  return row!;
}

export async function updatePool(
  poolId: string,
  data: Partial<{
    name: string;
    description: string | null;
    status: "active" | "paused" | "depleted" | "revoked";
    availableFrom: string | null;
    availableTo: string | null;
    availableDays: number[];
    maxTokensPerSession: number;
    maxTokensPerDay: number;
    allowedUseCases: string[];
    allowedModels: string[];
    creditsPerInputKToken: number;
    creditsPerOutputKToken: number;
    maxConcurrentSessions: number;
  }>
) {
  const db = getDb();
  const result = await db
    .update(tokenPool)
    .set(data)
    .where(eq(tokenPool.id, poolId))
    .returning();
  return result[0] ?? null;
}

export async function deletePool(poolId: string) {
  const db = getDb();
  const result = await db
    .delete(tokenPool)
    .where(eq(tokenPool.id, poolId))
    .returning();
  return result.length > 0;
}
```

### 3.3 Credit (`packages/data-ops/src/queries/credit.ts`)

```ts
import { eq } from "drizzle-orm";
import { getDb } from "../database/setup";
import { creditBalance, creditLedger } from "../drizzle/schema";

export const SIGNUP_BONUS_CREDITS = 500;

export async function getCreditBalance(userId: string) {
  const db = getDb();
  const result = await db
    .select()
    .from(creditBalance)
    .where(eq(creditBalance.userId, userId));
  return result[0] ?? null;
}

export async function createCreditBalance(userId: string, initialCredits: number) {
  const db = getDb();
  const [balance] = await db
    .insert(creditBalance)
    .values({ userId, available: initialCredits, reserved: 0 })
    .onConflictDoNothing()
    .returning();
  return balance ?? null;
}

export async function createLedgerEntry(data: {
  id: string;
  userId: string;
  amount: number;
  type: "signup_bonus" | "provider_earning" | "consumer_spend" | "refund";
  sessionId?: string | null;
  balanceAfter: number;
  description?: string | null;
}) {
  const db = getDb();
  const [entry] = await db.insert(creditLedger).values(data).returning();
  return entry!;
}
```

---

## 4. API Routes

### 4.1 Auth Middleware (Better Auth Session)

Add session-based auth middleware to `apps/data-service/src/hono/middleware/auth.ts`:

```ts
import { bearerAuth } from "hono/bearer-auth";
import type { MiddlewareHandler } from "hono";
import { getAuth } from "@repo/data-ops/auth/server";
import { HTTPException } from "hono/http-exception";

// Existing API token auth (keep)
export const authMiddleware = (token: string) =>
  bearerAuth({
    token,
    noAuthenticationHeaderMessage: "Authorization header required",
    invalidAuthenticationHeaderMessage: "Invalid authorization header format",
    invalidTokenMessage: "Invalid API key",
  });

// Session-based auth for user-facing endpoints
declare module "hono" {
  interface ContextVariableMap {
    userId: string;
  }
}

export const sessionAuth = (): MiddlewareHandler => {
  return async (c, next) => {
    const auth = getAuth();
    const session = await auth.api.getSession({
      headers: c.req.raw.headers,
    });
    if (!session?.user) {
      throw new HTTPException(401, { message: "Authentication required" });
    }
    c.set("userId", session.user.id);
    await next();
  };
};
```

### 4.2 Provider Config Routes (`apps/data-service/src/hono/handlers/provider-handlers.ts`)

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import {
  ProviderConfigCreateRequestSchema,
  ProviderConfigUpdateRequestSchema,
} from "@repo/data-ops/zod-schema/provider";
import { sessionAuth } from "../middleware/auth";
import * as providerService from "../services/provider-service";

const provider = new Hono<{ Bindings: Env }>();

provider.use("/*", sessionAuth());

// GET /api/provider/config
provider.get("/config", async (c) => {
  const userId = c.get("userId");
  return c.json(await providerService.getConfig(userId));
});

// POST /api/provider/config
provider.post(
  "/config",
  zValidator("json", ProviderConfigCreateRequestSchema),
  async (c) => {
    const userId = c.get("userId");
    const data = c.req.valid("json");
    return c.json(await providerService.createConfig(userId, data), 201);
  }
);

// PATCH /api/provider/config
provider.patch(
  "/config",
  zValidator("json", ProviderConfigUpdateRequestSchema),
  async (c) => {
    const userId = c.get("userId");
    const data = c.req.valid("json");
    return c.json(await providerService.updateConfig(userId, data));
  }
);

export default provider;
```

### 4.3 Pool Routes (`apps/data-service/src/hono/handlers/pool-handlers.ts`)

```ts
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import {
  PoolCreateRequestSchema,
  PoolUpdateRequestSchema,
  PoolIdParamSchema,
} from "@repo/data-ops/zod-schema/pool";
import { PaginationRequestSchema } from "@repo/data-ops/zod-schema/client";
import { sessionAuth } from "../middleware/auth";
import * as poolService from "../services/pool-service";

const pools = new Hono<{ Bindings: Env }>();

pools.use("/*", sessionAuth());

// GET /api/pools
pools.get("/", zValidator("query", PaginationRequestSchema), async (c) => {
  const userId = c.get("userId");
  const query = c.req.valid("query");
  return c.json(await poolService.listPools(userId, query));
});

// GET /api/pools/:id
pools.get("/:id", zValidator("param", PoolIdParamSchema), async (c) => {
  const userId = c.get("userId");
  const { id } = c.req.valid("param");
  const pool = await poolService.getPool(id, userId);
  return c.json(pool);
});

// POST /api/pools
pools.post(
  "/",
  zValidator("json", PoolCreateRequestSchema),
  async (c) => {
    const userId = c.get("userId");
    const data = c.req.valid("json");
    return c.json(await poolService.createPool(userId, data), 201);
  }
);

// PATCH /api/pools/:id
pools.patch(
  "/:id",
  zValidator("param", PoolIdParamSchema),
  zValidator("json", PoolUpdateRequestSchema),
  async (c) => {
    const userId = c.get("userId");
    const { id } = c.req.valid("param");
    const data = c.req.valid("json");
    return c.json(await poolService.updatePool(userId, id, data));
  }
);

// DELETE /api/pools/:id
pools.delete(
  "/:id",
  zValidator("param", PoolIdParamSchema),
  async (c) => {
    const userId = c.get("userId");
    const { id } = c.req.valid("param");
    await poolService.deletePool(userId, id);
    return c.body(null, 204);
  }
);

// POST /api/pools/:id/pause
pools.post(
  "/:id/pause",
  zValidator("param", PoolIdParamSchema),
  async (c) => {
    const userId = c.get("userId");
    const { id } = c.req.valid("param");
    return c.json(await poolService.pausePool(userId, id));
  }
);

// POST /api/pools/:id/resume
pools.post(
  "/:id/resume",
  zValidator("param", PoolIdParamSchema),
  async (c) => {
    const userId = c.get("userId");
    const { id } = c.req.valid("param");
    return c.json(await poolService.resumePool(userId, id));
  }
);

export default pools;
```

### 4.4 App Registration (`apps/data-service/src/hono/app.ts`)

```ts
import { Hono } from "hono";
import { requestId } from "./middleware/request-id";
import { createCorsMiddleware } from "./middleware/cors";
import { onErrorHandler } from "./middleware/error-handler";
import health from "./handlers/health-handlers";
import clients from "./handlers/client-handlers";
import provider from "./handlers/provider-handlers";
import pools from "./handlers/pool-handlers";

export const App = new Hono<{ Bindings: Env }>();

App.use("*", requestId());
App.onError(onErrorHandler);
App.use("*", createCorsMiddleware());

App.route("/health", health);
App.route("/clients", clients);
App.route("/api/provider", provider);
App.route("/api/pools", pools);
```

### 4.5 Route Summary

| Method | Path | Auth | Request Body | Response | Status |
|--------|------|------|-------------|----------|--------|
| GET | `/api/provider/config` | session | - | `ProviderConfigResponse` | 200 / 404 |
| POST | `/api/provider/config` | session | `ProviderConfigCreateRequest` | `ProviderConfigResponse` | 201 / 409 |
| PATCH | `/api/provider/config` | session | `ProviderConfigUpdateRequest` | `ProviderConfigResponse` | 200 / 404 |
| GET | `/api/pools` | session | query: `limit`, `offset` | `TokenPoolListResponse` | 200 |
| GET | `/api/pools/:id` | session | - | `TokenPoolResponse` | 200 / 404 / 403 |
| POST | `/api/pools` | session | `PoolCreateRequest` | `TokenPoolResponse` | 201 / 404 (no config) |
| PATCH | `/api/pools/:id` | session | `PoolUpdateRequest` | `TokenPoolResponse` | 200 / 404 / 403 |
| DELETE | `/api/pools/:id` | session | - | - | 204 / 404 / 403 |
| POST | `/api/pools/:id/pause` | session | - | `TokenPoolResponse` | 200 / 404 / 403 |
| POST | `/api/pools/:id/resume` | session | - | `TokenPoolResponse` | 200 / 404 / 403 |

### 4.6 Error Cases

| Scenario | Status | Body |
|----------|--------|------|
| No session cookie / invalid token | 401 | `{ "error": "Authentication required" }` |
| POST provider config when one exists | 409 | `{ "error": "Provider config already exists" }` |
| GET provider config when none exists | 404 | `{ "error": "Provider config not found" }` |
| POST pool without provider config | 404 | `{ "error": "Provider config not found" }` |
| PATCH/DELETE pool not owned by user | 403 | `{ "error": "Forbidden" }` |
| PATCH/DELETE pool not found | 404 | `{ "error": "Pool not found" }` |
| Pause already-paused pool | 409 | `{ "error": "Pool already paused" }` |
| Resume non-paused pool | 409 | `{ "error": "Pool is not paused" }` |
| Zod validation failure | 400 | `{ "error": "Validation failed", ... }` |

---

## 5. Services Layer

### 5.1 Provider Config Service (`apps/data-service/src/hono/services/provider-service.ts`)

```ts
import { HTTPException } from "hono/http-exception";
import {
  getProviderConfigByUserId,
  createProviderConfig as createProviderConfigQuery,
  updateProviderConfig as updateProviderConfigQuery,
} from "@repo/data-ops/queries/provider";
import type {
  ProviderConfigCreateInput,
  ProviderConfigUpdateInput,
  ProviderConfigResponse,
} from "@repo/data-ops/zod-schema/provider";
import { ulid } from "ulid";

function toResponse(row: Record<string, unknown>): ProviderConfigResponse {
  return {
    id: row.id as string,
    userId: row.userId as string,
    mode: row.mode as "local_proxy" | "api_key",
    llmProvider: row.llmProvider as string,
    modelAllowlist: row.modelAllowlist as string[],
    isActive: row.isActive as boolean,
    lastSeenAt: row.lastSeenAt ? (row.lastSeenAt as Date).toISOString() : null,
    reputationScore: row.reputationScore as number,
    totalSessionsServed: row.totalSessionsServed as number,
    totalTokensServed: row.totalTokensServed as number,
    createdAt: (row.createdAt as Date).toISOString(),
    updatedAt: (row.updatedAt as Date).toISOString(),
    hasApiKey: !!(row.encryptedApiKey),
  };
}

export async function getConfig(userId: string): Promise<ProviderConfigResponse> {
  const config = await getProviderConfigByUserId(userId);
  if (!config) throw new HTTPException(404, { message: "Provider config not found" });
  return toResponse(config);
}

export async function createConfig(
  userId: string,
  data: ProviderConfigCreateInput
): Promise<ProviderConfigResponse> {
  const existing = await getProviderConfigByUserId(userId);
  if (existing) throw new HTTPException(409, { message: "Provider config already exists" });

  // api_key mode blocked until Phase 5 encryption — only local_proxy allowed
  if (data.mode === "api_key") {
    throw new HTTPException(400, { message: "api_key mode not yet supported. Use local_proxy." });
  }

  const row = await createProviderConfigQuery({
    id: ulid(),
    userId,
    mode: data.mode,
    llmProvider: data.llmProvider ?? "claude",
    encryptedApiKey: null, // Phase 5: AES-256-GCM encryption
    modelAllowlist: data.modelAllowlist ?? [],
  });

  return toResponse(row);
}

export async function updateConfig(
  userId: string,
  data: ProviderConfigUpdateInput
): Promise<ProviderConfigResponse> {
  const existing = await getProviderConfigByUserId(userId);
  if (!existing) throw new HTTPException(404, { message: "Provider config not found" });

  const updateData: Record<string, unknown> = {};
  if (data.mode !== undefined) updateData.mode = data.mode;
  if (data.llmProvider !== undefined) updateData.llmProvider = data.llmProvider;
  if (data.modelAllowlist !== undefined) updateData.modelAllowlist = data.modelAllowlist;
  if (data.isActive !== undefined) updateData.isActive = data.isActive;
  if (data.mode === "api_key") {
    throw new HTTPException(400, { message: "api_key mode not yet supported. Use local_proxy." });
  }
  // apiKey field ignored until Phase 5 encryption

  const row = await updateProviderConfigQuery(existing.id, updateData);
  if (!row) throw new HTTPException(404, { message: "Provider config not found" });
  return toResponse(row);
}
```

### 5.2 Pool Service (`apps/data-service/src/hono/services/pool-service.ts`)

```ts
import { HTTPException } from "hono/http-exception";
import { getProviderConfigByUserId } from "@repo/data-ops/queries/provider";
import {
  getPoolById,
  getPoolsByProviderId,
  createPool as createPoolQuery,
  updatePool as updatePoolQuery,
  deletePool as deletePoolQuery,
} from "@repo/data-ops/queries/pool";
import type { PoolCreateInput, PoolUpdateInput } from "@repo/data-ops/zod-schema/pool";
import type { PaginationRequest } from "@repo/data-ops/zod-schema/client";
import { ulid } from "ulid";

async function requireProviderConfig(userId: string) {
  const config = await getProviderConfigByUserId(userId);
  if (!config) throw new HTTPException(404, { message: "Provider config not found" });
  return config;
}

async function requireOwnedPool(userId: string, poolId: string) {
  const config = await requireProviderConfig(userId);
  const pool = await getPoolById(poolId);
  if (!pool) throw new HTTPException(404, { message: "Pool not found" });
  if (pool.providerId !== config.id) throw new HTTPException(403, { message: "Forbidden" });
  return pool;
}

export async function listPools(userId: string, params: PaginationRequest) {
  const config = await getProviderConfigByUserId(userId);
  if (!config) return { data: [], pagination: { total: 0, limit: params.limit, offset: params.offset, hasMore: false } };
  return getPoolsByProviderId(config.id, params);
}

export async function getPool(poolId: string, userId: string) {
  const pool = await getPoolById(poolId);
  if (!pool) throw new HTTPException(404, { message: "Pool not found" });
  const config = await getProviderConfigByUserId(userId);
  if (!config || pool.providerId !== config.id) {
    throw new HTTPException(403, { message: "Forbidden" });
  }
  return pool;
}

// Serialize Drizzle row -> JSON-safe response (Date -> ISO string)
function poolToResponse(row: typeof tokenPool.$inferSelect): TokenPoolResponse {
  return {
    ...row,
    createdAt: row.createdAt.toISOString(),
    updatedAt: row.updatedAt.toISOString(),
  }
}

export async function createPool(userId: string, data: PoolCreateInput) {
  const config = await requireProviderConfig(userId);
  const row = await createPoolQuery({
    id: ulid(),
    providerId: config.id,
    name: data.name,
    description: data.description ?? null,
    availableFrom: data.availableFrom ?? null,
    availableTo: data.availableTo ?? null,
    availableDays: data.availableDays,
    maxTokensPerSession: data.maxTokensPerSession,
    maxTokensPerDay: data.maxTokensPerDay,
    allowedUseCases: data.allowedUseCases,
    allowedModels: data.allowedModels,
    creditsPerInputKToken: data.creditsPerInputKToken,
    creditsPerOutputKToken: data.creditsPerOutputKToken,
    maxConcurrentSessions: data.maxConcurrentSessions,
  });
  return poolToResponse(row);
}

export async function updatePool(userId: string, poolId: string, data: PoolUpdateInput) {
  await requireOwnedPool(userId, poolId);
  const row = await updatePoolQuery(poolId, data);
  if (!row) throw new HTTPException(404, { message: "Pool not found" });
  return poolToResponse(row);
}

export async function deletePool(userId: string, poolId: string) {
  await requireOwnedPool(userId, poolId);
  const deleted = await deletePoolQuery(poolId);
  if (!deleted) throw new HTTPException(404, { message: "Pool not found" });
}

export async function pausePool(userId: string, poolId: string) {
  const pool = await requireOwnedPool(userId, poolId);
  if (pool.status === "paused") throw new HTTPException(409, { message: "Pool already paused" });
  const row = await updatePoolQuery(poolId, { status: "paused" });
  if (!row) throw new HTTPException(404, { message: "Pool not found" });
  return poolToResponse(row);
}

export async function resumePool(userId: string, poolId: string) {
  const pool = await requireOwnedPool(userId, poolId);
  if (pool.status !== "paused") throw new HTTPException(409, { message: "Pool is not paused" });
  const row = await updatePoolQuery(poolId, { status: "active" });
  if (!row) throw new HTTPException(404, { message: "Pool not found" });
  return poolToResponse(row);
}
```

### 5.3 Credit Initialization on Signup

Use Better Auth's `databaseHooks` to create credit balance when a new user is created. Update `packages/data-ops/src/auth/setup.ts`:

```ts
import { betterAuth, type BetterAuthOptions } from "better-auth";
import {
  createCreditBalance,
  createLedgerEntry,
  SIGNUP_BONUS_CREDITS,
} from "../queries/credit";
import { ulid } from "ulid";

export const createBetterAuth = (config: {
  database: BetterAuthOptions["database"];
  secret?: BetterAuthOptions["secret"];
}): ReturnType<typeof betterAuth> => {
  return betterAuth({
    database: config.database,
    secret: config.secret,
    emailAndPassword: {
      enabled: true,
    },
    databaseHooks: {
      user: {
        create: {
          after: async (user) => {
            const balance = await createCreditBalance(
              user.id,
              SIGNUP_BONUS_CREDITS
            );
            if (balance) {
              await createLedgerEntry({
                id: ulid(),
                userId: user.id,
                amount: SIGNUP_BONUS_CREDITS,
                type: "signup_bonus",
                balanceAfter: SIGNUP_BONUS_CREDITS,
                description: "Welcome bonus",
              });
            }
          },
        },
      },
    },
  });
};
```

The `onConflictDoNothing()` in `createCreditBalance` ensures idempotency.

### 5.4 ULID Dependency

```bash
pnpm --filter @repo/data-ops add ulid
pnpm --filter data-service add ulid
```

---

## 6. Migration Plan

### 6.1 Steps

1. Add all tables + enums to `packages/data-ops/src/drizzle/schema.ts`
2. Update `packages/data-ops/src/drizzle/relations.ts`
3. Generate migration:
   ```bash
   cd packages/data-ops
   pnpm run drizzle:dev:generate
   ```
4. Review generated SQL -- verify enums, constraints, indexes
5. Run migration:
   ```bash
   pnpm run drizzle:dev:migrate
   ```
6. Verify tables exist:
   ```sql
   SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';
   ```
7. Build data-ops:
   ```bash
   pnpm --filter @repo/data-ops build
   ```

### 6.2 Staging/Production

Same flow, swap `dev` for `staging` or `production`:
```bash
pnpm run drizzle:staging:generate
pnpm run drizzle:staging:migrate
```

---

## 7. File Inventory

New files to create:

| File | Package |
|------|---------|
| `src/zod-schema/provider.ts` | data-ops |
| `src/zod-schema/pool.ts` | data-ops |
| `src/zod-schema/credit.ts` | data-ops |
| `src/queries/provider.ts` | data-ops |
| `src/queries/pool.ts` | data-ops |
| `src/queries/credit.ts` | data-ops |
| `src/hono/handlers/provider-handlers.ts` | data-service |
| `src/hono/handlers/pool-handlers.ts` | data-service |
| `src/hono/services/provider-service.ts` | data-service |
| `src/hono/services/pool-service.ts` | data-service |

Files to modify:

| File | Change |
|------|--------|
| `packages/data-ops/src/drizzle/schema.ts` | Add 6 tables + 5 enums |
| `packages/data-ops/src/drizzle/relations.ts` | Add all relations |
| `packages/data-ops/src/auth/setup.ts` | Add `databaseHooks` for credit init |
| `apps/data-service/src/hono/app.ts` | Register provider + pool routes |
| `apps/data-service/src/hono/middleware/auth.ts` | Add `sessionAuth()` middleware |
| `apps/data-service/.dev.vars` | Add `BETTER_AUTH_SECRET` |

---

## 8. Verification Checklist

Each item independently testable via curl or DB query.

### Auth & Session

- [ ] `GET /api/provider/config` without session cookie returns 401
- [ ] `POST /api/pools` without session cookie returns 401

### Provider Config

- [ ] `POST /api/provider/config` with valid body returns 201, persists to DB
- [ ] `POST /api/provider/config` with `mode: "api_key"` and no `apiKey` returns 400
- [ ] `POST /api/provider/config` when config already exists returns 409
- [ ] `GET /api/provider/config` returns config with `hasApiKey: true/false` (never exposes raw key)
- [ ] `GET /api/provider/config` when no config exists returns 404
- [ ] `PATCH /api/provider/config` with valid body returns 200, DB updated
- [ ] `PATCH /api/provider/config` with empty body returns 400

### Token Pools

- [ ] `POST /api/pools` with valid body returns 201, pool persisted
- [ ] `POST /api/pools` without provider config returns 404
- [ ] `POST /api/pools` with empty `allowedModels` returns 400
- [ ] `GET /api/pools` returns only pools owned by authenticated user
- [ ] `GET /api/pools?limit=2&offset=0` paginates correctly
- [ ] `GET /api/pools` returns empty array when user has no provider config (not 404)
- [ ] `PATCH /api/pools/:id` updates specified fields, leaves others unchanged
- [ ] `PATCH /api/pools/:id` for pool owned by different user returns 403
- [ ] `PATCH /api/pools/:id` for nonexistent pool returns 404
- [ ] `DELETE /api/pools/:id` returns 204, pool removed from DB
- [ ] `DELETE /api/pools/:id` for pool owned by different user returns 403
- [ ] `POST /api/pools/:id/pause` sets status to "paused", returns updated pool
- [ ] `POST /api/pools/:id/pause` on already-paused pool returns 409
- [ ] `POST /api/pools/:id/resume` sets status to "active", returns updated pool
- [ ] `POST /api/pools/:id/resume` on active pool returns 409

### Credit Initialization

- [ ] New user signup creates `credit_balance` row with `available: 500, reserved: 0`
- [ ] New user signup creates `credit_ledger` entry with `type: "signup_bonus"`, `amount: 500`, `balance_after: 500`
- [ ] Duplicate credit_balance insert (idempotency) does not error or double credits

### Database

- [ ] All 6 tables exist with correct columns and types
- [ ] All enums exist: `provider_mode`, `pool_status`, `session_status`, `session_end_reason`, `credit_type`
- [ ] `provider_config.user_id` has unique index
- [ ] `token_pool.provider_id` has FK to `provider_config.id` with cascade delete
- [ ] `credit_balance.user_id` is PK and FK to `auth_user.id` with cascade delete
- [ ] `credit_ledger.session_id` is nullable FK to `llm_session.id`

---

## 9. Resolved Questions

- ~~Store raw API key in Phase 1 or block `api_key` mode until encryption in Phase 5?~~ **RESOLVED: block `api_key` mode until Phase 5**
- ~~`PaginationRequestSchema` shared from `client.ts` -- extract to common zod-schema file?~~ **RESOLVED: keep in `client.ts` for MVP. Extract when needed by 3+ consumers.**
- ~~`ulid` vs `crypto.randomUUID()` for IDs -- PLAN says ulid but existing `clients` table uses uuid~~ **RESOLVED: `crypto.randomUUID()` — already used in existing tables, no extra dep.**
- ~~Should `DELETE /api/pools/:id` cascade-delete future sessions or reject if active sessions exist?~~ **RESOLVED: reject if active sessions exist (status=`active`). Completed/expired sessions cascade-delete.**
- ~~`available_days` uses 0=Sun..6=Sat (JS Date.getDay) or 1=Mon..7=Sun (ISO 8601)?~~ **RESOLVED: 0=Sun JS convention (Date.getDay)**
- ~~Better Auth `getSession` call per request -- cache in middleware or acceptable overhead?~~ **RESOLVED: acceptable overhead for MVP. Cache in middleware later if profiling shows bottleneck.**
