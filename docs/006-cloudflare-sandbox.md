# Phase 6 — Cloudflare Sandbox (Post-MVP)

> **POST-MVP. Do not plan sub-phases yet. CF Containers still in beta.**

> **Status:** Not started
> **Depends on:** Phases 1-5
> **Ref:** [PLAN.md](./PLAN.md) Phase 6

## Overview

Cloud-hosted Claude Code containers with org context pre-injected. Employee opens web terminal, gets a container with `.claude/CLAUDE.md` (generated from vision store) and `.mcp.json` (pre-configured with their token). Workspace persists via R2. No local Claude Code install needed. BYOK still applies.

## Architecture

```
Browser (xterm.js Web Terminal)
  ↕ WebSocket
CF Worker (sandbox-orchestrator)
  ├─ spawns CF Container per session
  ├─ injects CLAUDE.md + .mcp.json
  ├─ syncs workspace to/from R2
  └─ manages container lifecycle
  ↕ service binding
Hono API Worker (existing)

CF Container (per session)
  ├─ Claude Code CLI (BYOK)
  ├─ .claude/CLAUDE.md       # auto-generated from vision store
  ├─ .mcp.json               # pre-configured aigent MCP
  ├─ /workspace/             # R2-backed persistent workspace
  └─ Node.js + Git + common tools

R2 Bucket (workspace-storage)
  └─ workspaces/{orgId}/{userId}/workspace.tar.gz
```

## Features

- **Container per session** — CF Container with Claude Code CLI, spawned/stopped by orchestrator Worker
- **Context injection** — vision docs compiled into `.claude/CLAUDE.md`, `.mcp.json` auto-configured with user's token
- **R2 workspace persistence** — workspace tarball snapshot on stop, restore on start, periodic snapshots while running
- **Web terminal** — xterm.js in browser, WebSocket proxied through orchestrator to container PTY
- **API key isolation** — Anthropic key passed as container env var, never stored, gone on stop
- **Idle timeout** — auto-stop after inactivity, 1 container per user per org
- **Network egress restriction** — container can only reach MCP server, Anthropic API, npm registry

## Container Concept

CF Containers: Docker-based, started by Worker via `container` binding, share network with parent Worker (localhost), ephemeral by default, billed per container-second + memory.

Single Dockerfile in `apps/sandbox-container/` — Node.js base, Claude Code CLI, git. Entrypoint writes config files, restores workspace from R2, starts terminal multiplexer.

## R2 Workspace

Bucket: `aigent-workspaces`

```
workspaces/{orgId}/{userId}/workspace.tar.gz    # latest snapshot
workspaces/{orgId}/{userId}/metadata.json       # last modified, size
```

Limits: 500MB compressed max, 30-day retention after last access, R2 lifecycle rule for cleanup.

## Web Terminal

```
Browser (xterm.js) ↔ WebSocket ↔ Orchestrator Worker ↔ localhost TCP ↔ CF Container PTY
```

UI: API key input (sessionStorage only), start/stop, status indicator, full-screen resizable terminal.

## Security

- API key never stored — container env var only, ephemeral
- Org membership required to spawn
- 1 active container per user per org
- R2 encrypted at rest (default)
- Network egress restricted

## Unresolved Questions

- CF Containers GA timeline? API surface may change
- WebSocket support through Worker proxy to container PTY?
- Container cold start time — need pre-warmed pool?
- Max container runtime limits from CF?
- Can container binding pass env vars at spawn time or only deploy time?
- Workspace snapshots: incremental (rsync-style) vs full tarball?
- Per-user encryption for R2 workspaces worth the complexity?
- Container egress filtering: CF network policies or custom iptables?
- Pricing model: included in seat price or separate metered billing?
- Multiple concurrent terminals (tmux) or single session?

---