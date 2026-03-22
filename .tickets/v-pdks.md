---
id: v-pdks
status: open
deps: []
links: []
created: 2026-03-22T10:24:45Z
type: bug
priority: 1
assignee: Jibles
tags:
    - cli
    - balance
    - tokens
---
# CLI: Fix balance --include-tokens --fiat crash and inconsistent token sets

## Objective

`vasig balance --include-tokens --fiat` crashes non-deterministically with "Token pricing not supported for Solana (non-EVM chain)" and silently drops unpriceable tokens when it does succeed. Different flag combinations also return different token sets for the same wallet. Both issues make the balance command unreliable for agents.

## Context & Findings

### Non-deterministic crash

`vault.balancesWithPrices(chains, includeTokens, 'usd')` throws when the vault has discovered SPL tokens (e.g., PUMP on Solana). The error originates in the SDK's pricing service. Sometimes the SDK catches it internally and drops the token; sometimes it propagates.

The SDK fix is tracked in `vc-mpnw`, but the CLI should wrap the call with per-chain error handling so a pricing failure on one chain doesn't crash the entire command.

### Silent token dropping

When `balancesWithPrices` succeeds, Solana's PUMP token vanishes from output with no warning. An agent making portfolio decisions misses assets entirely.

### Inconsistent token sets across code paths

Three internal paths return different tokens for the same wallet:

- `--include-tokens` (no fiat): uses `vault.balances()` → shows USDC, PEPE, PUMP, Arbitrum USDC, AVAX
- `--chain Ethereum --include-tokens --fiat`: uses `vault.balance()` per chain → shows USDC, PEPE, USDT
- `--include-tokens --fiat` (all chains): uses `vault.balancesWithPrices()` → shows USDC, PEPE only

USDT appears only in the single-chain fiat path. PUMP appears only in the non-fiat path.

### Rejected approach

Don't try to unify the three SDK methods — they have legitimately different behaviors. Instead, catch pricing failures per-chain and include unpriceable tokens with `fiatValue` omitted.

## Files

- `src/commands/balance.ts` — `getBalances()` has three code paths (lines 35-82). The `opts.fiat` path needs try/catch around `balancesWithPrices`. On failure, fall back to `vault.balances()` for that chain and include tokens without fiat values.

## Acceptance Criteria

- [ ] `vasig balance --include-tokens --fiat` never crashes — pricing failures are caught per-chain
- [ ] Tokens that can't be priced are included in output with `fiatValue` omitted (not silently dropped)
- [ ] `vasig balance --chain Solana --include-tokens --fiat` returns SOL with fiat + PUMP without fiat (not crash)
- [ ] Works correctly in both table and JSON output modes
- [ ] Lint and type-check pass

## Gotchas

- Root cause is in the SDK (`vc-mpnw`) — our fix is a defensive workaround
- Per-chain fallback: if `balancesWithPrices` throws, iterate chains individually with `vault.balance()` and skip fiat for that chain's tokens
- USDT appearing only in single-chain path is because the per-chain path calls `vault.balance(chain, tokenId)` for each persisted token, while `balancesWithPrices` only includes tokens the SDK discovers internally — document for now
