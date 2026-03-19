---
id: tic-cedf
status: open
type: task
priority: 2
assignee: Jibles
tags:
    - chat-app
    - sdk
    - refactor
deps:
    - tic-1302
    - tic-8306
    - tic-8980
created: 2026-03-19T02:48:44.86563707Z
---
# Chat App: Rewire tool handlers to use SDK

## Gotchas

- The SDK's `sign()` method handles the entire MPC flow internally (relay setup, WASM keysign, signature compilation) — don't try to replicate any of that in the app
- SDK methods are async — the orchestrator already handles async tool execution, but verify the event emitter patterns still work
- Some tool handlers do input transformation (e.g., resolving chain names from strings, looking up coins by symbol) — this logic may need to stay in the app if the SDK doesn't handle fuzzy matching
- `AgentContextService` currently builds context by calling multiple storage functions — switching to SDK means the data shapes may differ slightly. Map carefully.
- The agent backend (agent.vultisig.com) sends tool call names that must match exactly — don't rename tools
- Password/confirmation prompts are UI concerns and stay in the app — only the crypto execution moves to SDK

## Acceptance Criteria

- [ ] Zero imports from `@core/chain/`, `@core/mpc/`, `@lib/dkls`, `@lib/schnorr`, `@trustwallet/wallet-core`
- [ ] `@vultisig/sdk` is the only crypto dependency
- [ ] All 40 tool handlers work end-to-end through the SDK
- [ ] Signing flow: prepareTx → sign → broadcast works via SDK
- [ ] Vault authentication uses SDK's unlock + session token flow
- [ ] Context builder provides vault data via SDK methods
- [ ] Tool handler files are each <50 lines (thin wrappers)
- [ ] The orchestrator's total crypto-related code is <200 lines
- [ ] No WASM files bundled in the chat app
- [ ] App bundle size is significantly smaller (no chain/mpc code)
- [ ] Integration tests: at minimum send + swap + balance flows work
- [ ] Lint and type-check pass

