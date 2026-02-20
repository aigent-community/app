# aigent.community

Shared context layer for dev teams using AI coding agents. Boss sets vision/rules → MCP server serves it to every employee's Claude Code → agents report activity back → boss sees alignment dashboard.

**BYOK** — platform never touches Anthropic API. Employees use their own Claude subscriptions.

## How It Works

1. Boss creates org → writes vision (rules, decisions, goals)
2. Employee adds MCP server to `.mcp.json`
3. Agent calls `get_organization_context` → gets org vision
4. Agent works → **must** call `report_activity` with summary
5. Boss sees activity feed → flags misalignment
6. Learnings propagate to all agents via shared memory

## Architecture

```
Employee's Claude Code (BYOK)
  ↕ MCP protocol (SSE)
Remote MCP Server (CF Worker)
  ├─ get_organization_context
  ├─ get_shared_memory
  ├─ search_decisions
  ├─ report_activity
  └─ get_recent_activities
  ↕ service binding
Hono API Worker → Neon PostgreSQL
TanStack Start SSR Worker → Dashboard UI
```

## Tech Stack

Built on [saas-on-cf](https://github.com/AuditMos/saas-on-cf) by Auditmos.

| Layer | Tech |
|-------|------|
| Frontend | React 19 + TanStack Start (SSR on CF Workers) |
| API | Hono on Cloudflare Workers |
| MCP | Remote MCP server (SSE transport on CF Workers) |
| Database | Drizzle ORM + Neon Postgres |
| Auth | Better Auth (email/pwd, Google OAuth later) |

## Monorepo

```
apps/
  data-service/           # REST API + MCP server (Hono on CF Workers)
  user-application/       # SSR frontend (TanStack Start)

packages/
  data-ops/               # Shared DB layer (Drizzle, Zod, Better Auth)
```

## Development

```bash
pnpm run setup                    # install + build data-ops
pnpm run dev:user-application     # frontend (port 3000)
pnpm run dev:data-service         # API (port 8788)
```

## Deployment

```bash
pnpm run deploy:staging:user-application
pnpm run deploy:staging:data-service
pnpm run deploy:production:user-application
pnpm run deploy:production:data-service
```

## Implementation Phases

| Phase | Scope | Status |
|-------|-------|--------|
| 0 | Landing page | |
| 1 | Schema + Auth + Org — tables, CRUD, member invites, API tokens | |
| 2 | MCP Server — 5 tools, SSE transport, token auth, rate limiting | |
| 3 | Vision + Activity UI — editor, activity feed | |
| 4 | Dashboard + Memory — alignment overview, memory browser, CLAUDE.md export | |
| 5 | Polish — versioning, filters, full-text search, self-host guide | |
| 6 | Cloudflare Sandbox — cloud containers (post-MVP) | |

## Design Docs

Full specs in `/docs`:

```
docs/
├── PLAN.md                  # Master plan (single source of truth)
├── 000-landing.md
├── 001-schema-auth-org.md
├── 002-mcp-server.md
├── 003-vision-activity-ui.md
├── 004-dashboard-alignment.md
├── 005-polish.md
└── 006-cloudflare-sandbox.md
```

## License

Proprietary.
