---
id: v-cojg
status: closed
deps: []
links: []
created: 2026-03-22T10:24:46Z
type: bug
priority: 1
assignee: Jibles
tags:
    - cli
    - chains
    - tokens
    - state
---
# CLI: State mutation bugs — add-all drops extra chains, discover inflates extraChains

## Objective

Two separate bugs cause unexpected mutations to the vault's persisted state in `~/.vultisig/config.json`. `chains --add-all` drops previously-added extra chains, and `tokens --discover` silently adds chains to `extraChains` even when no tokens are found on them.

## Context & Findings

### Bug 1: `chains --add-all` drops Arbitrum

After running `vasig chains --add-all`, the active chain count was 35 instead of 36. Arbitrum — which was an "extra chain" persisted in config — was missing from the active list.

**Root cause:** In `src/commands/chains.ts`, the `--add-all` block (around line 33) snapshots `vault.chains` before calling `vault.setChains([...SUPPORTED_CHAINS])`, then computes `newExtras` as chains in SUPPORTED_CHAINS that weren't in the original snapshot. But Arbitrum was already in `vault.chains` (loaded from extraChains), so it's NOT in `newExtras`. Then `persistExtraChains(vaultEntry.id, newExtras)` **replaces** the existing extraChains with only the newly-added ones, dropping Arbitrum.

**Fix:** `newExtras` should be the union of existing `currentExtras` and newly-discovered extras, not just the new ones. Or simpler: after `setChains`, persist all non-default chains as extras.

### Bug 2: `tokens --discover` inflates extraChains

Running `vasig tokens --discover` followed by `vasig tokens --clear` cycles caused `extraChains` to grow from `["Arbitrum"]` to 17 chains including Dash, Zcash, Cosmos, etc.

**Root cause:** `tokens --discover` iterates `vault.chains` (line 20 of tokens.ts). But the discover path in `balance.ts` (auto-discover on first `--include-tokens`) calls `vault.discoverTokens(chain)` which may trigger `vault.addChain()` internally if the chain isn't in the vault. These added chains then get persisted to `extraChains` via `createSdkWithVault()`'s chain restoration logic on the next invocation, or through the separate `persistExtraChains` call.

Actually, looking more carefully: the issue is that `tokens --discover` without `--chain` uses `[...vault.chains]` which is correct — it only scans active chains. But the auto-discover path in `balance.ts` may be the culprit, or the SDK's `discoverTokens` may add chains internally. Needs investigation.

**Fix:** Token discovery should never modify `extraChains`. If `discoverTokens()` internally adds chains, those additions should not be persisted. Only `vasig chains --add` should modify `extraChains`.

## Files

- `src/commands/chains.ts` — fix `--add-all` block to preserve existing extraChains
- `src/commands/balance.ts` — ensure auto-discover doesn't persist chain additions
- `src/commands/tokens.ts` — ensure `--discover` doesn't persist chain additions
- `src/auth/config.ts` — reference for `persistExtraChains()`

## Acceptance Criteria

- [x] `vasig chains --add-all` preserves all previously-active chains (including extra chains from config)
- [x] After `--add-all`, active count equals `SUPPORTED_CHAINS.length` (currently 36)
- [x] `vasig tokens --discover` does not modify `extraChains` in config (confirmed: SDK discoverTokens/addToken don't mutate chain state)
- [x] `vasig balance --include-tokens` auto-discover does not modify `extraChains` in config (confirmed: no persistExtraChains calls in balance.ts)
- [x] Multiple discover/clear cycles leave `extraChains` unchanged
- [x] Lint and type-check pass

## Gotchas

- The `--add-all` fix is straightforward: change `persistExtraChains(vaultEntry.id, newExtras)` to include `currentExtras` that are still valid
- The discover/extraChains issue needs investigation — verify whether it's the CLI code or the SDK's `discoverTokens()` that adds chains. Add a snapshot of `vault.chains` before discover and compare after to confirm.
- Don't break the intentional chain persistence that `vasig chains --add` uses
