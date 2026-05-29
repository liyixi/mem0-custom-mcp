# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

- **Build**: `npm run build` — TypeScript compilation via `tsc`
- **Dev run**: `npm run dev` — build + run the server locally
- **Start**: `npm start` — run the compiled `dist/index.js`
- **Install**: `npm install` (no lockfile regeneration needed; use `npm ci` in CI)
- **CI locally**: `npm ci && npm run build && test -f dist/index.js`

There is no test suite, linter, or formatter configured.

## Architecture

This is an MCP stdio server (Node.js/TypeScript ESM) that wraps a self-hosted Mem0 HTTP API so Claude Code can use it for memory management. A single file — [src/index.ts](src/index.ts) — contains the entire implementation.

**Flow**: Claude Code → MCP stdio → this server → HTTP REST → Self-Hosted Mem0 API → PostgreSQL/Neo4j

- **Transport**: `StdioServerTransport` from `@modelcontextprotocol/sdk` — all logging goes to stderr to avoid corrupting the stdio channel.
- **API helper**: `callMem0API(endpoint, method, body)` — all Mem0 HTTP calls go through this. It has a 120-second `AbortController` timeout (line 59) because Mem0's embedding/LLM processing can take 30-60 seconds.
- **Tool dispatch**: single `CallToolRequestSchema` handler with a `switch(name)` block for the 4 tools. Each branch validates with Zod, then calls `callMem0API`.
- **Internal version is hardcoded** as `"1.0.0"` in the `Server` constructor (line 84) — this is NOT synced with `package.json` version (currently 1.1.0).

**Tools and their Mem0 API endpoints**:

| Tool | HTTP call |
|------|-----------|
| `add_memory` | `POST /v1/memories/` — body wraps content in `messages[{role:"user", content}]` |
| `search_memories` | `POST /v1/memories/search/` |
| `get_memories` | `GET /v1/memories/{user_id}` — then client-side `.slice(0, limit)` |
| `delete_memory` | `DELETE /v1/memories/{memory_id}` |

**Configuration**: `MEM0_API_URL` (default `http://localhost:8888`), `DEFAULT_USER_ID` (default `"default"`).

## CI / Publishing

- **CI** (`.github/workflows/ci.yml`): builds on Node 18 and 20, verifies `dist/index.js` exists.
- **Publish** (`.github/workflows/publish.yml`): manual `workflow_dispatch` with version input — bumps version, publishes to npm with `--provenance --access public`, then pushes the tag and commit.
