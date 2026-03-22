---
id: v-pvgw
status: open
deps: []
links: []
created: 2026-03-22T10:24:48Z
type: task
priority: 2
assignee: Jibles
tags:
    - cli
    - agent
    - json
---
# CLI: Agent JSON output — fiat as number, missing fields, specific error codes

## Objective

The CLI's JSON output (`--output json`) has several issues that make it harder for agents to consume: fiat values are formatted strings instead of numbers, send/swap responses are missing key fields, and error codes are generic `UNKNOWN_ERROR` instead of specific types. Fix these to make the CLI reliably machine-parseable.

## Context & Findings

### fiatValue is a string, not a number

```json
"fiatValue": "$3.57"
```

An agent must parse the `$` prefix to do arithmetic. This should be a raw number (`3.57`) with the currency conveyed separately via `fiatCurrency`. Same issue for `estimatedOutputFiat` in swap quotes.

This affects: `balance --fiat`, `swap-quote`, and the pre-swap quote summary in `swap`.

### Send/swap responses missing key fields

Send JSON response:
```json
{ "txHash": "0x...", "chain": "Arbitrum", "explorerUrl": "https://..." }
```

Missing: `amount`, `to`, `token`, `symbol`. An agent can't confirm what was sent without re-querying.

Swap quote missing: `inputFiat` (agent can't compare input vs output value).

Balance missing when `--include-tokens`: `contractAddress`, `decimals` per token entry.

### Generic error codes

All non-`VasigError` exceptions get `"code": "UNKNOWN_ERROR"`. Known error types that should have specific codes:
- Invalid chain → `INVALID_CHAIN`
- Invalid address → `INVALID_ADDRESS`
- Insufficient balance → `INSUFFICIENT_BALANCE`
- No swap route → `NO_ROUTE`
- Token not found → `TOKEN_NOT_FOUND`
- Pricing unavailable → `PRICING_UNAVAILABLE`

## Files

- `src/commands/balance.ts` — change `fiatValue` from `$${x.toFixed(2)}` to raw number
- `src/commands/swap.ts` — change `estimatedOutputFiat` from `$${x.toFixed(2)}` to raw number, add `inputFiat` to quote
- `src/commands/send.ts` — add `amount`, `to`, `token` to send result
- `src/types.ts` — update `BalanceResult.fiatValue` type from `string` to `number`, update `SwapQuoteResult`, update `SendResult`
- `src/lib/errors.ts` — add new error classes or update error wrapping to use specific codes
- `src/lib/output.ts` — update table formatting to add `$` prefix for display when value is a number (table mode only)

## Acceptance Criteria

- [ ] `fiatValue` in balance JSON is a number (e.g., `3.57` not `"$3.57"`)
- [ ] `fiatCurrency` remains as `"USD"` string alongside numeric `fiatValue`
- [ ] Table mode still shows `$3.57` for human readability (format in output layer)
- [ ] `estimatedOutputFiat` in swap-quote JSON is a number
- [ ] Swap quote includes `inputFiat` field (numeric)
- [ ] Send response includes `amount`, `to`, `symbol` fields
- [ ] Balance with `--include-tokens` includes `contractAddress` and `decimals` per entry
- [ ] At least 5 specific error codes replace `UNKNOWN_ERROR` for known error types
- [ ] Lint and type-check pass

## Gotchas

- Changing `fiatValue` from string to number is a breaking change for any existing consumers — since this is pre-release (v0.1.0), acceptable
- The `$` prefix formatting for table mode needs to move from the data layer (`balance.ts`) to the output layer (`output.ts` or inline in `balanceCommand`) — or just format in the table rendering path
- The send response `amount` should be the human-readable amount string that was passed in, not the raw bigint
