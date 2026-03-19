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

## Critical: Credential Collection Without LLM Exposure

Passwords must NEVER flow through the LLM/agent layer. The MCP server communicates with Claude Code via stdio JSON-RPC — any password passed as a tool parameter would be visible to the model. Two side-channel mechanisms are required:

### Primary: CLI pre-authentication (recommended)

User runs `npx @vultisig/cli auth` before or during a Claude Code session:

```
$ npx @vultisig/cli auth
  Vault file: ~/.vultisig/my-vault.vult
  Local password: ••••••••
  Server password: ••••••••

  ✓ Vault unlocked
  ✓ Session token cached (~/.vultisig/session.json, expires 24h)
```

The MCP server on startup reads `~/.vultisig/session.json` (contains server session token + reference to vault file). It decrypts the local key share using a key derived during the CLI auth step. The LLM never sees any credentials.

### Fallback: localhost browser callback

If no session file exists when the MCP server starts:
1. MCP server starts a temporary HTTP server on a random localhost port
2. Returns a tool result: "Authentication required — opening browser..."
3. Opens `http://localhost:{port}/auth` with a form: vault path, local password, server password
4. User submits form → credentials go directly to MCP server process via localhost POST
5. MCP server authenticates, caches session, shuts down HTTP server
6. Subsequent tool calls proceed normally

This mirrors `gh auth login` and OAuth device flows. The browser form talks directly to the MCP process — credentials never enter the stdio/LLM channel.

### For the chat app (different flow)

The chat app is a React UI — it has a dedicated unlock screen that collects passwords and passes them directly to `sdk.loadVault()` and `sdk.authenticate()` in-process. Passwords are held in SDK memory only, never serialized to React state, conversation context, or the agent backend. The agent orchestrator receives an already-unlocked vault instance.

### Security invariants

- No MCP tool schema accepts a password parameter — the LLM cannot request credentials
- Passwords exist in memory only during initial auth, then are discarded (replaced by session token + decrypted key share)
- `~/.vultisig/session.json` must be chmod 600 (owner-only)
- Session token is scoped to one vault and expires in 24h
- On token expiry, tools return an actionable error ("Run `npx @vultisig/cli auth` to re-authenticate") — no automatic re-prompting through the LLM

## Gotchas

- MCP stdio servers must not write anything to stdout except JSON-RPC — all logging must go to stderr
- The `@modelcontextprotocol/sdk` server class handles the transport — don't implement JSON-RPC manually
- If server session token endpoint (tic-ef85) is not yet available, fall back to the CLI auth step holding the server password encrypted on disk with a local-only key
- WASM libraries (DKLS, Schnorr) need to work in Node.js — the SDK already handles this via its node build target
- Tool names should use snake_case to match MCP conventions
- Consider adding a `vault_info` tool that shows vault name, type, chains, and signer count — useful for agent orientation
- The localhost callback server must bind to 127.0.0.1 only (not 0.0.0.0) to prevent network exposure

## Acceptance Criteria

- [ ] `@vultisig/mcp` package builds and publishes from the monorepo
- [ ] MCP server starts via stdio and registers all tools
- [ ] At minimum these tools work end-to-end: `get_balances`, `get_address`, `send`, `swap`, `get_portfolio`
- [ ] CLI auth: `npx @vultisig/cli auth` prompts for passwords and writes session file
- [ ] MCP server reads session file on startup — no password prompting through tool calls
- [ ] Fallback: localhost browser auth flow works when no session file exists
- [ ] No tool schema contains a password or credential field
- [ ] Sensitive operations (sign, send, swap) require confirmation step
- [ ] Claude Code can discover and use the server via `claude mcp add` config
- [ ] README documents setup: `claude mcp add vultisig -- npx @vultisig/mcp`
- [ ] All tool inputs have JSON Schema definitions
- [ ] Error responses are clear and actionable (not raw stack traces)
- [ ] `~/.vultisig/session.json` created with 600 permissions
- [ ] `yarn check:all` passes in the monorepo

