---
id: v-kdfq
status: open
deps: []
links: []
created: 2026-03-22T10:24:49Z
type: chore
priority: 2
assignee: Jibles
tags:
    - cli
    - validation
    - errors
---
# CLI: Input validation and error message cleanup

## Objective

Multiple commands accept invalid inputs without local validation, producing confusing errors from the SDK or remote APIs. Error messages leak raw SDK/library internals (viem stack traces, LI.FI SDK version strings). Silent no-ops on remove/duplicate-add make agent automation unreliable. Clean all of this up.

## Context & Findings

### Case-sensitive chain names with no help

`vasig balance --chain ethereum` fails with: `Unknown chain: "ethereum". Available: [Arbitrum, Base, ...]` — dumps the full 36-chain list but doesn't suggest "Did you mean: Ethereum?". Case-insensitive matching would be the simplest fix.

### Invalid chain names give wrong errors

`vasig chains --add NonexistentChain` returns "Password required for encrypted vault" — the code tries to derive an address before validating the chain name. Should validate against `SUPPORTED_CHAINS` first.

### Verbose SDK errors leak through

- `vasig tokens --add 0xinvalid --chain Ethereum` → full viem `ContractFunctionExecutionError` with call arguments
- `vasig swap-quote --from Arbitrum --to Arbitrum --amount 0.001` → LI.FI SDK version string and validation details
- `vasig swap-quote --amount abc` → raw viem NaN error

These should be wrapped in human-friendly messages.

### Silent no-ops

- `vasig tokens --remove 0xnonexistent --chain Ethereum` → reports `action: removed`, exit 0 (nothing was removed)
- `vasig chains --add Avalanche` when Avalanche is already active → reports `action: added`, exit 0 (no change)
- `vasig chains --remove NonexistentChain` → reports `action: removed`, exit 0

Agents can't distinguish success from no-op.

### Memo breaks EVM sends

`vasig send --chain Arbitrum --memo "test"` → `eth_sendTransaction format error`. Either encode memo in tx data field or reject with "memos not supported on EVM chains".

## Files

- `src/commands/chains.ts` — validate chain name against `SUPPORTED_CHAINS` before `vault.addChain()`, report `already-active`/`not-found` for duplicates/missing
- `src/commands/tokens.ts` — validate chain name, check token exists before remove, wrap SDK errors
- `src/commands/balance.ts` — case-insensitive chain matching
- `src/commands/swap.ts` — validate amount locally (> 0, numeric), wrap SDK errors for same-chain and other known failures
- `src/commands/send.ts` — handle `--memo` on EVM chains (reject or encode), wrap SDK errors
- `src/lib/errors.ts` — optionally add helper for wrapping SDK errors with friendlier messages

## Acceptance Criteria

- [ ] Chain names are matched case-insensitively across all commands (balance, chains, tokens, send, swap)
- [ ] `chains --add NonexistentChain` returns a clear "not a supported chain" error (not "Password required")
- [ ] `chains --add` on an already-active chain returns `action: already-active` or similar (not `action: added`)
- [ ] `chains --remove` on an inactive chain returns `action: not-found` (not `action: removed`)
- [ ] `tokens --remove` on a non-existent token returns an error or `action: not-found`
- [ ] SDK/library errors (viem, LI.FI) are wrapped in clean messages — no stack traces, URLs, or version strings in user-facing output
- [ ] `send --memo` on EVM chains either works (hex-encoded in data) or returns "memos not supported on EVM chains"
- [ ] All error responses use appropriate error codes (not `UNKNOWN_ERROR`) in JSON mode
- [ ] Lint and type-check pass

## Gotchas

- Case-insensitive matching: the SDK uses exact `Chain` enum values (e.g., `"THORChain"`, `"BSC"`). Match user input case-insensitively against `SUPPORTED_CHAINS`, then use the canonical value for SDK calls.
- For EVM memo: the SDK's `prepareSendTx` accepts a `memo` field — check if it encodes it properly or if we need to hex-encode ourselves. If the SDK handles it, the bug may be in how we pass it.
- Wrapping SDK errors: catch known error patterns (e.g., messages containing "ContractFunctionExecutionError", "LI.FI", "Invalid address") and replace with clean messages. Preserve the original in a debug/verbose mode if needed.
