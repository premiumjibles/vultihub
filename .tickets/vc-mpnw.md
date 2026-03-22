---
id: vc-mpnw
status: open
deps: []
links: []
created: 2026-03-22T09:10:56Z
type: task
priority: 2
assignee: Jibles
tags:
    - sdk
    - signing
    - agent
---
# SDK: Signing reliability and agent-friendliness improvements

## Context

During real-world testing of the vultiagent-cli (send/swap operations across Arbitrum, BSC, Solana, THORChain, Ethereum), we identified two categories of issues in `@vultisig/sdk` v0.5.0 that affect agent consumers:

1. **MPC keysign reliability** — first signing attempt frequently times out, second succeeds immediately
2. **Console logging** — SDK logs progress to stdout with emojis, polluting structured CLI output

## Issue 1: MPC Keysign First-Attempt Timeout

### Observed Behavior

The first `vault.sign()` call in a session consistently times out after 1 minute. A retry after ~15 seconds succeeds in seconds. This pattern is reproducible across all chains (Arbitrum, BSC, Ethereum, etc.).

The signing progress shows everything is set up correctly — server joins, participants ready, MPC session starts — then hangs at the actual keysign step until the 1-minute abort fires.

### Root Cause Analysis

A race condition between the SDK client's 1-minute timer and VultiServer's startup pipeline:

1. SDK calls `POST /vault/sign` → server enqueues an asynq task and returns immediately
2. SDK joins relay, finds peers, starts MPC session, uploads setup message
3. SDK starts a **hard-coded 1-minute `AbortController` timeout** inside `keysign()`
4. VultiServer is still processing: picking up the Redis task → loading/decrypting vault from disk → polling relay for session start (1s intervals) → polling for setup message (1s intervals) → deserializing keyshare via CGo FFI → creating SignSession
5. Server-side latency from "session started" to "ready for MPC messages" can consume 5-30+ seconds
6. The actual multi-round DKLS signing doesn't have enough time in the remaining window → timeout

**Why retry works immediately:** WASM is warm (10.3MB `vs_wasm_bg.wasm` compiled on first call, cached via `memoizeAsync`), server has vault decrypted and CGo lib loaded, HTTP/TLS connections pooled.

### Recommended Fixes (ordered by impact)

**A. Pre-warm WASM on SDK init** (high impact, easy)
Call `Re("ecdsa")` during `Vultisig` constructor or `importVault()` rather than lazily inside `keysign()`. This removes WASM compilation from the signing critical path.

**B. Increase keysign timeout** (high impact, easy)
The 1-minute timeout is too tight given server-side latency. VultiServer's own timeout is `3*time.Minute + 3*time.Second`. SDK should match — 2-3 minutes minimum, or make it configurable:
```typescript
vault.sign({ ..., timeout: 180_000 })
```

**C. Don't start timer until server is ready** (high impact, medium effort)
Instead of starting the 1-minute timer immediately after `ensureSetupMessage()`, poll a relay indicator that confirms the server has fetched the setup message before beginning the countdown.

**D. Reduce server polling intervals** (medium impact, easy)
`WaitForSessionStart` and `WaitForSetupMessage` in VultiServer use 1-second backoff between polls. Reducing to 100-200ms would save several seconds of dead time.

**E. Expose timeout configuration** (low effort, enables workarounds)
Currently the keysign timeout is hard-coded. Exposing it in the `sign()` options would let consumers increase it without SDK changes.

### Additional Issue: ERC-20 Approve + Swap Double-Sign

When `prepareSwapTx` is called with `autoApprove: false`, the SDK sets `approvalPayload = undefined` in the return value, but leaves `erc20ApprovePayload` embedded in the main `keysignPayload`. This causes `extractMessageHashes` → `getEvmSigningInputs` to produce 2 message hashes, signed sequentially in `coordinateFastSigning`. Each gets its own 1-minute timeout, but the server-side session state can go stale between the two signs.

**Fix:** Either strip `erc20ApprovePayload` from the keysign payload when `autoApprove: false`, or sign multiple message hashes in parallel rather than sequentially.

## Issue 2: Console Logging Pollutes Agent Output

### Observed Behavior

During signing, the SDK logs directly to stdout:
```
🔑 Generated signing party ID: sdk-6935
📡 Calling FastVault API with session ID: b351d96e-...
✅ Server acknowledged session: b351d96e-...
⏳ Waiting for server to join session...
✅ All participants ready: [sdk-6935, Server-253]
📡 Starting MPC session with devices list...
✅ MPC session started
🔐 Starting MPC keysign process...
🔏 Signing message: c18fd349...
✅ Signature obtained for message
🔄 Formatting signature results...
```

These lines are hardcoded `console.log` calls in the SDK bundle. They mix with structured CLI output (JSON/table), making it difficult for agents to parse responses cleanly.

### Problem for Agents

- Pollutes stdout when CLI uses `--output json` — agents get emoji text interleaved with JSON
- Adds ~10 lines of noise to agent context windows per signing operation
- Contains session IDs and message hashes that are useless to consumers
- No way to suppress without patching the SDK

### Recommended Fix

The SDK already has a proper `signingProgress` event system (`vault.on('signingProgress', ...)`) with structured `SigningStep` objects. The console logging should be removed or gated behind a debug/verbose flag:

**Option A (preferred):** Remove all `console.log` calls from the signing path. Consumers who want progress can subscribe to `signingProgress` events.

**Option B:** Add a `logLevel` or `verbose` option to the `Vultisig` constructor:
```typescript
const sdk = new Vultisig({ logLevel: 'silent' })  // suppress console output
```

**Option C:** Route signing logs through `console.debug` instead of `console.log`, so they're hidden by default and only visible when `NODE_DEBUG` or similar is enabled.

## CLI Workaround (Current State)

The vultiagent-cli currently works around Issue 1 with `signWithRetry()` — 1 retry after 15s delay on timeout errors. This is safe because the timeout occurs during MPC keysign (before broadcast), so no transaction is signed or sent on the failed attempt.

There is no workaround for Issue 2 without patching the SDK.

## Issue 3: `balancesWithPrices()` Throws for Non-EVM Token Pricing

### Observed Behavior

`vault.balancesWithPrices(chains, includeTokens, 'usd')` throws `"Token pricing not supported for Solana (non-EVM chain)"` when the vault has discovered SPL tokens (e.g., PUMP on Solana). The error is non-deterministic — sometimes it throws, sometimes it silently drops the unpriceable tokens and returns the rest.

### Impact

CLI's `vasig balance --include-tokens --fiat` crashes intermittently. The CLI cannot reliably show fiat values for all assets when tokens are included.

### Expected Behavior

Should never throw for unsupported token pricing. Instead, return the token with `fiatValue: null` or `fiatValue: 0` and let the consumer decide how to handle it.

## Issue 4: Swap Quote Providers Return Near-Zero Output Amounts

### Observed Behavior

Several swap routes via thorchain/mayachain providers return estimates that are essentially zero (10^-12 or less) but the SDK returns them as valid quotes:

- `Ethereum → Arbitrum` (0.01 ETH): estimatedOutput = `0.00000000000098376 ETH` ($0.00) via mayachain
- `Ethereum → BSC` (0.01 ETH): estimatedOutput = `0.000000000003242353 BNB` ($0.00) via thorchain
- `Bitcoin → Ethereum` (0.00001 BTC): estimatedOutput = `0.000000000000020819 ETH` ($0.00) via thorchain

The same routes work correctly via li.fi (returning reasonable amounts).

### Impact

If an agent executes a swap based on these quotes, it loses essentially all funds. The CLI returns `ok: true` for these quotes because the SDK doesn't validate output reasonableness.

### Expected Behavior

The SDK should either:
1. Try multiple providers and return the best quote
2. Reject quotes where output value is near-zero relative to input value
3. Flag the quote with a warning when output/input ratio is extremely unfavorable

## Acceptance Criteria

- [ ] First `vault.sign()` call in a fresh session succeeds reliably (no timeout on cold start)
- [ ] Keysign timeout is configurable, or increased to 2+ minutes
- [ ] WASM modules are pre-initialized during SDK/vault setup, not lazily in `keysign()`
- [ ] Console logging during signing is suppressible (via option, log level, or removal)
- [ ] `signingProgress` events are the primary progress mechanism, not console output
- [ ] ERC-20 approve + swap flow doesn't produce stale server sessions with `autoApprove: false`
- [ ] `getSupportedSwapChains()` returns deduplicated chain list (currently returns duplicates from multiple providers)
- [ ] `balancesWithPrices()` gracefully handles non-EVM token pricing (returns null/0 instead of throwing)
- [ ] Swap quote provider selection rejects near-zero output amounts, or tries alternative providers
- [ ] Solana outbound signing works reliably (currently 0% success rate — large tx payloads ~500+ bytes cause consistent MPC timeouts)
- [ ] Solana blockhash refresh on retry — when MPC signing times out and retries, the Solana blockhash in the keysign payload may have expired, causing broadcast failure even after successful retry signing
- [ ] Overall MPC first-attempt success rate improved (currently ~33% across 2 rounds of testing, ~30 signing attempts)

## Notes

**2026-03-22T11:25:09Z**

## SDK Opportunities from CLI Validation Work (v-kdfq)

The CLI (vasig) is working around several things the SDK should own:

- **`resolveChain()`** — case-insensitive chain name resolution with "did you mean?" suggestions. Currently implemented in vasig's lib/validation.ts, but every SDK consumer needs this. Should live in the SDK alongside SUPPORTED_CHAINS.
- **Chain metadata** — vasig hardcodes an EVM_CHAINS list to reject memos on EVM chains. SUPPORTED_CHAINS should export chain metadata (isEVM, supportsMemo, nativeToken, etc.) so consumers don't maintain their own lists.
- **Clean error messages** — vasig's classifyError() cleans up viem/LI.FI error dumps into human-friendly messages. The SDK should throw clean errors at the source instead of leaking library internals.
