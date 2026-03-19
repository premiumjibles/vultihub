---
id: tic-c6cc
status: open
type: feature
priority: 1
assignee: Jibles
tags:
    - mcp
    - sdk
    - claude-code
deps:
    - tic-8306
    - tic-8980
    - tic-ef85
created: 2026-03-19T02:47:24.067924215Z
---
# MCP: Create @vultisig/mcp server package

## Gotchas

- MCP stdio servers must not write anything to stdout except JSON-RPC — all logging must go to stderr
- The `@modelcontextprotocol/sdk` server class handles the transport — don't implement JSON-RPC manually
- Session token caching: `~/.vultisig/session.json` must be 600 permissions (owner-only read/write)
- If server session token endpoint (tic-ef85) is not yet available, fall back to holding server password in memory for the process lifetime
- WASM libraries (DKLS, Schnorr) need to work in Node.js — the SDK already handles this via its node build target
- Tool names should use snake_case to match MCP conventions
- Consider adding a `vault_info` tool that shows vault name, type, chains, and signer count — useful for agent orientation

## Acceptance Criteria

- [ ] `@vultisig/mcp` package builds and publishes from the monorepo
- [ ] MCP server starts via stdio and registers all tools
- [ ] At minimum these tools work end-to-end: `get_balances`, `get_address`, `send`, `swap`, `get_portfolio`
- [ ] Auth flow: prompts for vault path + passwords on first use, caches session token
- [ ] Sensitive operations (sign, send, swap) require confirmation step
- [ ] Claude Code can discover and use the server via `claude mcp add` config
- [ ] README documents setup: `claude mcp add vultisig -- npx @vultisig/mcp`
- [ ] All tool inputs have JSON Schema definitions
- [ ] Error responses are clear and actionable (not raw stack traces)
- [ ] `yarn check:all` passes in the monorepo

