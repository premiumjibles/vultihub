---
id: v-fzjh
status: closed
deps: []
links: []
created: 2026-03-22T10:24:43Z
type: bug
priority: 0
assignee: Jibles
tags:
    - cli
    - safety
    - swap
    - send
---
# CLI: Swap/send safety — quote sanity check and pre-flight balance validation

## Objective

Prevent fund loss from bad swap quotes and wasted time from insufficient-balance sends. Currently the CLI accepts near-zero swap quotes as valid (`ok: true`) and allows sends that exceed balance to proceed all the way to MPC signing before failing with a cryptic timeout.

## Context & Findings

### Near-zero swap quotes (fund loss risk)

Several routes via thorchain/mayachain providers return estimates of ~10^-12 tokens but the CLI presents them as valid:

- `Ethereum → Arbitrum` (0.01 ETH → 0.00000000000098376 ETH, $0.00) via mayachain
- `Ethereum → BSC` (0.01 ETH → 0.000000000003242353 BNB, $0.00) via thorchain
- `Bitcoin → Ethereum` (0.00001 BTC → 0.000000000000020819 ETH, $0.00) via thorchain

The same routes via li.fi return reasonable amounts. The SDK provider selection is the root cause (tracked in `vc-mpnw`), but the CLI **must** have its own safety net regardless.

The `estimatedOutputFiat` field is available from the SDK and already surfaced in quotes. When it's `$0.00` on a non-trivial input, the CLI should reject.

### No balance validation before send

`vasig send --chain Arbitrum --amount 999 --yes` (far exceeding the ~0.002 ETH balance) proceeds to MPC signing, waits 60s for timeout, retries, waits another 60s, then fails with `NETWORK_ERROR: Fast signing failed: Exited inbound processing due to a timeout after 1 min`. Total wasted time: ~2 minutes. The user never sees "insufficient balance".

The balance is already fetched at the start of `executeSend()` (line 23: `vault.balance(opts.chain, opts.token)`). We just need to compare.

### Missing local input validation

- `vasig swap --from Arbitrum --to Arbitrum --amount 0.001` — same-chain swap hits the remote API instead of failing locally
- `vasig swap-quote --amount 0` — zero amount hits remote API instead of local rejection
- `vasig swap-quote --amount -1` — negative amount hits remote API
- `vasig swap-quote --amount abc` — non-numeric produces raw viem error

## Files

- `src/commands/swap.ts` — add quote sanity check in `executeSwap()` after `getSwapQuote()`, add local input validation in `buildSwapQuoteParams()`
- `src/commands/send.ts` — add balance comparison after `vault.balance()` in `executeSend()`
- `src/commands/swap.ts` — add input validation for amount (> 0, numeric) and same-chain check in `getSwapQuote()` and `executeSwap()`

## Acceptance Criteria

- [ ] Swap quotes with `estimatedOutputFiat` of `$0.00` (or null) on input > 0 are rejected with a clear error: "Quote output is near-zero — this route would result in fund loss"
- [ ] Swap quotes where output/input fiat ratio is < 0.5 (50%+ loss) emit a warning; < 0.01 (99%+ loss) are blocked unless `--yes`
- [ ] `vasig send` with amount > balance fails immediately with "Insufficient balance: you have X [symbol], tried to send Y" before any signing attempt
- [ ] Same-chain swaps (`--from X --to X`) are rejected locally: "Cannot swap within the same chain"
- [ ] Zero and negative amounts are rejected locally: "Amount must be greater than 0"
- [ ] Non-numeric amounts are rejected locally: "Amount must be a valid number"
- [ ] All validations return proper exit codes and work in both table and JSON output modes
- [ ] Lint and type-check pass

## Gotchas

- Balance comparison for sends must account for decimals — the amount is parsed to bigint in `executeSend()`, compare against `balance.amount` (also bigint) or use the SDK's raw balance
- The `estimatedOutputFiat` may be undefined when `fiatCurrency` wasn't requested in the quote — the CLI already passes `fiatCurrency: 'usd'` in `buildSwapQuoteParams()` so this should always be present
- Don't block legitimate low-value swaps — a $0.01 → $0.005 swap is a 50% loss but might be intentional for dust. The near-zero check (< 0.01 ratio) is the critical one
