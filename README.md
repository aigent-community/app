# aigent.community

LLM token sharing marketplace. Subscription owners share unused Claude tokens with others via real-time chat sessions. Internal credit economy — no real money changes hands.

## How It Works

**Providers** share their LLM access by creating token pools with custom limits (models, daily caps, time windows, pricing in credits).

**Consumers** browse the marketplace, pick a pool, start a chat session. Credits are reserved upfront, settled on session end.

Two proxy modes:
- **Local proxy** — provider runs CLI agent, API key never leaves their machine
- **API key delegation** — provider stores encrypted key on platform (Phase 5+)

## Architecture

```
Browser (Consumer/Provider)
  | WebSocket + REST
TanStack Start SSR Worker ──service binding──> Hono API Worker
                                                ├─ SessionAgent (per session, Cloudflare Agents SDK)
                                                ├─ ProviderAgent (per provider, Cloudflare Agents SDK)
                                                ├─ KV (pool cache, online status)
                                                ├─ Queues (usage-log, credit-tx)
                                                └─ Neon PostgreSQL
Provider CLI Agent
  | WebSocket
ProviderAgent
```

### Session Flow

1. Consumer creates session → credits reserved
2. Consumer connects via WebSocket → SessionAgent initialized
3. Messages route through LLM proxy (API key or local proxy mode)
4. Response streams back as chunks with live token counters
5. Budget warnings at 80% and 95%
6. Session end → credits settled, provider earns credits

## Tech Stack

Built on [saas-on-cf](https://github.com/AuditMos/saas-on-cf) template by Auditmos.

| Layer | Tech |
|-------|------|
| Frontend | React 19 + TanStack Start (SSR on CF Workers) |
| API | Hono on Cloudflare Workers |
| Realtime | Cloudflare Agents SDK (Durable Objects) + WebSocket |
| Database | Drizzle ORM + Neon Postgres |
| Auth | Better Auth |
| Queues | Cloudflare Queues (usage logging, credit processing) |
| CLI | Node.js (commander + ws) |

## Monorepo Structure

```
apps/
  user-application/       # SSR frontend (TanStack Start)
  data-service/           # REST API + Agents (Hono on CF Workers)
  cli/                    # Provider CLI agent (Node.js)

packages/
  data-ops/               # Shared DB layer (Drizzle, Zod, Better Auth)
```

## Key Entities

| Entity | Purpose |
|--------|---------|
| `provider_config` | Provider settings — mode, model allowlist, reputation |
| `token_pool` | Shareable token bucket — limits, pricing, schedule |
| `llm_session` | Active/completed chat session between consumer & pool |
| `credit_balance` | User's available + reserved credits |
| `credit_ledger` | Full credit transaction history |
| `usage_log` | Per-request token usage metrics |

## Development

```bash
pnpm run setup                    # install + build data-ops
pnpm run dev:user-application     # frontend (port 3000)
pnpm run dev:data-service         # API (port 8788)
```

### Database

```bash
# from packages/data-ops/
pnpm run drizzle:dev:generate     # generate migration
pnpm run drizzle:dev:migrate      # apply migration
```

Replace `dev` with `staging` or `production` for other environments.

### Environment Variables

Config in `packages/data-ops/`:
- `.env.dev` — local development
- `.env.staging` — staging
- `.env.production` — production

See [.env.example](./packages/data-ops/.env.example) for required values.

## Deployment

```bash
# Staging
pnpm run deploy:staging:user-application
pnpm run deploy:staging:data-service

# Production
pnpm run deploy:production:user-application
pnpm run deploy:production:data-service
```

## Implementation Phases

| Phase | Scope | Status |
|-------|-------|--------|
| 1 | Schema + Auth — Drizzle tables, provider/pool CRUD, credit init | |
| 2 | Core Session — Agents, chat UI, LLM proxy, token metering, credits | |
| 3 | Marketplace + Dashboards — browse pools, provider/consumer dashboards, queues | |
| 4 | CLI Agent + Local Proxy — CLI scaffolding, WS client, LLM forwarding | |
| 5 | Polish — budget warnings, export, rate limiting, encryption, reputation | |

## Design Docs

Full specifications live in `/docs`:

```
docs/
├── PLAN.md                        # Master plan
├── 001-schema-and-auth/           # DB schema, Zod, queries, API routes, migrations
├── 002-core-session/              # SessionAgent, LLM proxy, credit engine, WS protocol, chat UI
├── 003-marketplace-and-dashboards/# Marketplace API, credits API, queues, dashboard UI
├── 004-cli-agent-local-proxy/     # CLI scaffold, WS client, message router, auth tokens
└── 005-polish/                    # Warnings, export, time windows, rate limits, encryption
```

## License

Proprietary.
