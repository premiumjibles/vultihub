---
id: v-dcci
status: open
deps: []
links: []
created: 2026-03-22T13:54:31Z
type: task
priority: 2
assignee: Jibles
---
# Secure Vault Signing Support in vultiagent-cli — Research Brief

# Secure Vault Signing Support in vultiagent-cli

## Problem Statement
The vultiagent-cli currently only supports fast vault signing (2-of-2 with VultiServer). Users with secure vaults (no server key share — all shares on their own devices) cannot use the CLI at all. This is especially limiting when Claude Code drives the CLI autonomously, where interactive elements like QR codes are impossible. We need the CLI to initiate a secure signing session and get user approval on their phone without requiring visual or interactive elements.

## Research Findings

### Current State
- CLI signing flow (`src/lib/signing.ts`, `src/commands/send.ts`, `src/commands/swap.ts`) calls `vault.sign()` which goes through the SDK's FastVault path — 2-of-2 MPC with VultiServer
- `src/auth/credential-store.ts` stores a "server password" for VultiServer auth — no concept of relay-based cosigning
- `src/commands/auth.ts:96` prompts explicitly for VultiServer password
- The CLI doesn't check `vault.type === 'fast'` — it just assumes VultiServer. Loading a secure vault would fail at the SDK level
- Vault files (`.vult`) are discovered from standard paths (`src/auth/vault-discovery.ts`) and can be either fast or secure — the CLI just can't do anything useful with secure ones today

### Available Tools & Patterns
- **Relay server** (`api.vultisig.com/router`) — already handles all session management:
  - `POST /{sessionID}` — register party
  - `GET /start/{sessionID}` — poll for all parties joined
  - `POST /message/{sessionID}` — send encrypted MPC messages
  - `POST /complete/{sessionID}` — mark completion
  - Messages encrypted with AES-256-CBC using a shared hex encryption key
- **Deep links** — both iOS and Android already handle: `https://vultisig.com?type=SignTransaction&vault=<pubKeyECDSA>&jsonData=<base64_proto>`
  - The QR code data IS this URL — so a link works just as well as a QR code
  - `jsonData` contains a protobuf-encoded `KeysignMessage` with: sessionID, encryptionKeyHex, keysignPayload, serviceName, useVultisigRelay flag
- **Push notifications** — already wired up:
  - iOS: APNS via `PushNotificationManager`, device tokens stored in Keychain
  - Android: FCM via `VultisigFirebaseMessagingService`
  - Device registration: `DeviceRegistrationRequest` with vaultId (SHA256 of pubKeyECDSA + hexChainCode), partyName, token, deviceType
  - Notification payloads already include `deeplink` in userInfo — tapping joins the keysign session automatically
  - `ForegroundNotificationParser` extracts vault pubKey and transaction type from the deep link
- **MPC threshold** — `ceil(N × 2/3)` signers needed. For a 2-device secure vault, both must participate. For 3 devices, 2 of 3.
- **Mobile join flow** — fully implemented: notification/QR/deep link → extract sessionID + encryptionKeyHex → register with relay → poll for start → exchange MPC messages → complete

### Validated Behaviors
- Fast vault identification: any vault with a signer whose party ID starts with "Server-" (case insensitive)
- Secure vault: no "Server-" signer — all signers are user device IDs
- The relay server is protocol-agnostic — it routes encrypted messages between any parties. It doesn't care if one party is a CLI
- KeysignMessage proto contains everything the joiner needs: sessionID, encryptionKeyHex, payload, relay flag
- Both DKLS and GG20 TSS protocols are used for signing — the specific protocol depends on when the vault was created

## Constraints
- SDK constraint: `@vultisig/sdk` may only expose FastVault signing internally — SecureVault relay signing may not be accessible from the current SDK API. This is the biggest unknown.
- Threshold constraint: for a 2-signer secure vault, BOTH devices must participate — there's no "approve on one side" shortcut
- Encryption: all relay messages must be encrypted with the session's shared AES key — the CLI must generate this and include it in the deep link/notification
- Proto format: the KeysignMessage must be serialized as protobuf, compressed, and base64-encoded to match what mobile apps expect
- Push notification: requires the vault to have a registered device token — if the user hasn't enabled notifications on their phone, this path won't work

## Dead Ends
- **Telegram integration**: no existing Telegram bot infrastructure in the codebase. Only TON chain references. Would need to be built from scratch.
- **Two key shares on one machine** (Option E): technically trivial but defeats the entire security model of secure vaults. Not worth pursuing as a primary path.
- **QR codes for Claude Code context**: impossible — Claude Code has no visual output channel for QR codes. Even in interactive terminal use, QR rendering is fragile.

## Explored Paths

### Path A: CLI + Phone via Push Notification (Recommended Primary)
CLI creates relay session → triggers push notification to phone with embedded deep link → user taps notification → phone joins session and cosigns → CLI polls for completion → broadcasts transaction.
- **Pros**: Best UX, no user action beyond tapping "approve". Works headlessly (Claude Code compatible).
- **Cons**: Requires push notifications to be enabled on the phone. Need to figure out how to trigger the push from CLI (may need a server-side endpoint).

### Path B: CLI + Phone via Deep Link (Recommended Fallback)
CLI creates relay session → outputs deep link URL to stdout → user opens link on phone → phone joins and cosigns.
- **Pros**: Simple, no extra infrastructure. Works in interactive terminal sessions.
- **Cons**: Doesn't work when Claude Code is driving (no human watching stdout). Manual step.

### Path C: CLI + Phone via Configurable Webhook (Recommended for Headless)
CLI creates relay session → sends join link via user-configured channel (Telegram bot, Slack webhook, email, etc.) → user opens link on phone → cosigns.
- **Pros**: Works headlessly. User chooses their notification channel. Extensible.
- **Cons**: Requires setup (bot token, chat ID, webhook URL). New feature to build.
- Config could look like: `{ "secureSign": { "notify": "telegram", "telegramBotToken": "...", "telegramChatId": "..." } }`

### Path D: Two CLI Instances (Cosigner Mode)
Run a persistent `cosign` listener on a second machine with its own key share. First CLI initiates, second auto-approves (or prompts).
- **Pros**: Fully automated for server deployments. No phone needed.
- **Cons**: Requires two machines. More ops overhead. Niche use case.

## Loose Recommendations
- The push notification path (A) should be the primary flow — the infrastructure already exists on the mobile side, and it provides the best UX for both interactive and Claude Code use
- Webhook/Telegram (C) is the best complement for users who haven't set up push notifications or want a different channel
- Deep link to stdout (B) should always be available as the zero-config fallback
- The SDK investigation should happen first — if SecureVault relay signing isn't exposed, that's the heaviest lift and gates everything else
- The CLI should auto-detect vault type on import and branch the signing flow accordingly, rather than requiring user configuration

## Open Questions
- Does `@vultisig/sdk` expose relay-based MPC signing for SecureVault, or only the FastVault server path? If not, what SDK changes are needed?
- Can the CLI trigger push notifications directly via an API, or does it need to go through a server-side endpoint? What's the notification service API?
- Should the webhook/Telegram notifier be built into the CLI core, or be a configurable hook/plugin?
- Timeout behavior: how long should the CLI wait for phone approval? Should it be configurable? What's the UX for timeout (retry prompt vs fail)?
- Auto-approval: should there be an option for the phone to auto-approve signing requests from a trusted CLI (reduces to fast-vault-like UX but without server trust)?
- Multi-transaction batching: if Claude Code is doing multiple operations, can signing requests be batched or does each need separate approval?
