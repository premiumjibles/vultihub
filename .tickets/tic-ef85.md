---
id: tic-ef85
status: open
type: feature
priority: 1
assignee: Jibles
tags:
    - server
    - auth
    - security
    - api.vultisig.com
created: 2026-03-19T02:46:44.717400857Z
---
# Server: Session token auth endpoint for vault signing

## Objective

Add a session token authentication endpoint to api.vultisig.com so that SDK/MCP clients can authenticate once with the vault password and receive a time-limited token for subsequent signing requests. This avoids storing or repeatedly prompting for the raw server-side password.

## User Story

An MCP server or CLI tool needs to sign multiple transactions in a session without prompting the user for their server password each time, and without persisting the raw password to disk.

## Design Constraints

- Must be backwards-compatible: existing `/vault/sign` must continue accepting raw password
- The token is for the server-side key share only — the local key share has its own password managed client-side
- Token should be short-lived (suggested: 24h) with explicit expiry
- Token should be scoped to a specific vault (public key)
- This ticket requires changes to the Vultisig server (api.vultisig.com) — NOT the SDK or client apps

## Context & Findings

**Current flow** (`vultisig-windows/core/ui/agent/tools/shared/fastVaultApi.ts`):
- `POST https://api.vultisig.com/vault/sign` accepts `vault_password` in the request body
- Password is sent in plaintext over HTTPS on every sign request
- No session concept exists — each request is stateless

**Proposed flow:**
1. `POST /vault/session` — accepts `{ public_key, vault_password }`, returns `{ token, expires_at }`
2. `POST /vault/sign` — accepts either `vault_password` OR `session_token` (backwards compatible)
3. Token is a signed JWT or opaque token scoped to the vault's public key
4. Server validates token on each `/vault/sign` call instead of re-decrypting the password

**Why this matters for MCP:**
- MCP servers are long-running processes — prompting for password per-sign is poor UX
- Storing raw password in env vars or config files is a security risk
- A session token with expiry is the standard pattern (cf. GitHub CLI `gh auth login`, AWS session tokens)
- The token can be stored in `~/.vultisig/session.json` with minimal risk — it expires and is vault-scoped

**Rejected: client-side password caching** — holding the raw password in memory for hours is less secure than a purpose-built session token with server-side validation and expiry.

## Files

Server-side (api.vultisig.com — not in this repo, reference only):
- New endpoint: `POST /vault/session`
- Modified endpoint: `POST /vault/sign` — add token validation path
- Token storage/validation infrastructure

SDK integration (will be done in ticket MCP-4, not here):
- `packages/sdk/src/server/FastSigningService.ts` — will add token-based auth path

## Acceptance Criteria

- [ ] `POST /vault/session` endpoint accepts `{ public_key, vault_password }` and returns `{ token, expires_at }`
- [ ] Token is valid for 24 hours (configurable)
- [ ] Token is scoped to the specific vault public key — cannot be used for other vaults
- [ ] `POST /vault/sign` accepts `{ session_token }` as alternative to `{ vault_password }`
- [ ] Existing password-based `/vault/sign` flow continues to work unchanged
- [ ] Invalid/expired tokens return 401 with clear error message
- [ ] Token is invalidated if vault password changes
- [ ] Rate limiting on `/vault/session` to prevent brute force (e.g., 5 attempts per minute per public key)

## Gotchas

- This is a server-side ticket — requires coordination with whoever maintains api.vultisig.com
- The token format (JWT vs opaque) affects whether the server needs a token store or can validate statelessly
- If using JWT: the vault password hash should NOT be in the token payload — use a server-side signing key
- Consider adding `DELETE /vault/session` for explicit logout/revocation
- The MCP and SDK tickets depend on this endpoint existing before they can implement token-based auth

