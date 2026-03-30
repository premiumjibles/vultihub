---
id: v-ugin
status: closed
deps: [v-wdyq]
links: [tic-ef85]
created: 2026-03-22T00:38:44Z
type: feature
priority: 1
assignee: Jibles
---
# MCP: Create @vultisig/mcp server package

## Objective

Create a stdio MCP (Model Context Protocol) server that exposes Vultisig vault operations as tools for AI agents like Claude Code. The server imports `@vultisig/sdk` directly and reads credentials from the same keyring/config that `vultiagent-cli` (`vasig auth`) sets up. This means the human authenticates once via the CLI, and the MCP server works automatically.

## User Story

A developer using Claude Code wants to check their crypto portfolio, send tokens, or execute swaps by chatting naturally. The MCP server makes Vultisig operations available as tools that Claude can call directly.

## Design Constraints

- Stdio transport only (spawned as subprocess by Claude Code) — no HTTP server
- Must not write anything to stdout except JSON-RPC messages — all logging goes to stderr
- Reads credentials from system keyring (set by `vasig auth`) — never prompts for passwords
- Uses `@vultisig/sdk` directly for all operations — does NOT shell out to the CLI
- Tool names use snake_case per MCP convention
- All tool inputs have JSON Schema definitions
- Errors are clear and actionable (not raw stack traces)
- Should be installable via: `claude mcp add vultisig -- npx @vultisig/mcp`

## Context & Findings

### Architecture: SDK-direct, not CLI wrapper

The MCP server imports `@vultisig/sdk` and calls vault methods directly (balance, send, swap, etc.). It does NOT wrap the CLI as a subprocess. Reasons:
- No subprocess overhead per tool call
- Native structured data (no JSON stdout parsing)
- Type safety between MCP tool schemas and SDK types
- The CLI and MCP are siblings that both consume the SDK — not parent/child

### Auth: Shared with CLI

The MCP server reads from the same credential sources as the CLI:
1. System keyring: `vultisig/<vaultId>/server` for server password, `vultisig/<vaultId>/decrypt` for vault decryption
2. `~/.vultisig/config.json` for vault file path and vault ID
3. `VAULT_PASSWORD` / `VAULT_DECRYPT_PASSWORD` env vars as fallback (useful for `claude mcp add` config)

If no credentials are found, the MCP server returns a tool error: "Vault not authenticated. Run `vasig auth` in a terminal first."

The credential-reading code should be extracted into a shared package or module that both the CLI and MCP import. This is the `auth/credential-store.ts` and `auth/config.ts` from the CLI ticket — either publish as a separate package or copy the module.

### MCP spec compliance

Per the MCP spec (modelcontextprotocol.io):
- Stdio transport: no MCP-level OAuth needed — server is a trusted local subprocess
- Credentials come from environment or local files/keyring
- `@modelcontextprotocol/sdk` handles JSON-RPC transport — don't implement manually

### Future: Session tokens

When tic-ef85 (server session tokens) lands, the MCP server benefits automatically — the credential store will return a session token instead of a password. No MCP-specific changes needed.

### Minimum tool set

| Tool | Description | SDK Method |
|------|-------------|------------|
| `get_balances` | Get token balances for a chain or all chains | `vault.balances()`, `vault.balance(chain)` |
| `get_portfolio` | Get portfolio with fiat values | `vault.balancesWithPrices()` |
| `get_address` | Get derived address for a chain | `vault.address(chain)` |
| `send` | Send tokens to an address | `vault.prepareSendTx()` → `vault.sign()` → `vault.broadcastTx()` |
| `swap` | Swap tokens between chains | `vault.getSwapQuote()` → `vault.prepareSwapTx()` → `vault.sign()` → `vault.broadcastTx()` |
| `swap_quote` | Get a swap quote without executing | `vault.getSwapQuote()` |
| `vault_info` | Show vault name, type, chains, signer count | Vault metadata |
| `supported_chains` | List chains the vault supports | `vault.getSupportedSwapChains()` |

### Confirmation for destructive operations

`send` and `swap` tools should include a two-step pattern:
1. First call returns a preview (amounts, fees, recipient) and asks the agent to confirm
2. Agent presents preview to user, gets confirmation
3. Second call with `confirmed: true` executes the transaction

This prevents accidental execution — the agent must explicitly confirm after seeing the preview.

## Files

New package (location TBD — could be in `premiumjibles/vultiagent-cli` monorepo or separate repo):
```
src/
  index.ts              — MCP server entry point (stdio transport)
  tools/
    balances.ts         — get_balances, get_portfolio tools
    addresses.ts        — get_address tool
    send.ts             — send tool (with confirmation step)
    swap.ts             — swap, swap_quote tools
    vault-info.ts       — vault_info, supported_chains tools
  auth/
    credential-store.ts — Shared with CLI: keyring + env var credential reading
    config.ts           — Shared with CLI: ~/.vultisig/config.json reading
  lib/
    errors.ts           — MCP error formatting
package.json
```

Reference:
- `vultisig-sdk/clients/cli/src/commands/swap.ts` — swap flow SDK usage
- `vultisig-sdk/clients/cli/src/commands/transaction.ts` — send flow SDK usage
- `vultisig-sdk/clients/cli/src/commands/balance.ts` — balance query SDK usage
- MCP SDK docs: https://modelcontextprotocol.io/docs

## Acceptance Criteria

- [ ] MCP server starts via stdio and registers all tools with JSON Schema definitions
- [ ] `get_balances`, `get_portfolio`, `get_address`, `vault_info`, `supported_chains` work end-to-end
- [ ] `send` and `swap` work end-to-end with two-step confirmation pattern
- [ ] `swap_quote` returns quote without executing
- [ ] Server reads credentials from keyring (set by `vasig auth`) without prompting
- [ ] Falls back to VAULT_PASSWORD env var if keyring unavailable
- [ ] Returns clear error if no credentials found ("Run vasig auth first")
- [ ] All logging goes to stderr, only JSON-RPC on stdout
- [ ] Claude Code can discover and use the server via `claude mcp add vultisig -- npx @vultisig/mcp`
- [ ] Error responses are structured and actionable
- [ ] Lint and type-check pass

## Gotchas

- MCP stdio servers must not write ANYTHING to stdout except JSON-RPC — console.log will break the protocol. Use console.error or a stderr logger.
- The `@modelcontextprotocol/sdk` server class handles transport — don't implement JSON-RPC manually
- WASM libraries (DKLS, Schnorr) load async — SDK must be initialized before registering tools
- The SDK's `Vultisig` constructor takes `onPasswordRequired` callback — wire to credential store, not a prompt
- `keytar` (keyring access) requires native modules — may need prebuild configuration for npx usage
- For the two-step send/swap confirmation: MCP doesn't have built-in confirmation — implement via tool input params (`confirmed: true/false`) or separate preview/execute tools
- The shared credential-store code between CLI and MCP should be a lightweight extraction, not a full shared package initially — can be promoted to a package later if needed

## Notes

**2026-03-23T02:06:22Z**

# Design

## Architecture

Single package — MCP server lives in `src/mcp/` within `vultiagent-cli`, invoked via `vasig mcp` subcommand. No separate package or build target.

```
┌──────────────┐   ┌──────────────┐
│  CLI (index)  │   │  MCP (vasig   │
│  Commander    │   │  mcp command) │
│  + output.ts  │   │  + tools.ts   │
└──────┬───────┘   └──────┬───────┘
       │                   │
       ▼                   ▼
┌─────────────────────────────────┐
│  commands/ (business logic fns) │
│  getBalances, executeSend, ...  │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│  auth/ + lib/ (shared infra)   │
│  credential-store, sdk, signing │
└──────────────┬──────────────────┘
               │
               ▼
┌─────────────────────────────────┐
│        @vultisig/sdk            │
└─────────────────────────────────┘
```

CLI command functions (getBalances, executeSend, etc.) are the shared logic layer. CLI wraps them with Commander + printResult(). MCP wraps them with JSON Schema tool defs + MCP response formatting. Neither layer knows about the other.

**New files:**
- `src/mcp/index.ts` — Server setup, stdio transport, vault initialization
- `src/mcp/tools.ts` — Tool registration (JSON Schema definitions + handler functions)

**New dependency:** `@modelcontextprotocol/sdk`

Minor cleanup may be needed in some command functions to ensure they return data cleanly rather than mixing in stderr output.

## Tools & Data Flow

8 tools mapping to existing CLI capabilities:

| Tool | Calls | Confirmation |
|------|-------|-------------|
| `get_balances` | `getBalances()` | No |
| `get_portfolio` | `getBalances()` with fiat flag | No |
| `get_address` | addresses.ts logic | No |
| `vault_info` | vault.ts logic | No |
| `supported_chains` | chains.ts logic | No |
| `swap_quote` | `getSwapQuote()` | No |
| `send` | `executeSend()` | Yes — `confirmed` flag |
| `swap` | `executeSwap()` | Yes — `confirmed` flag |

**Confirmation flow (send/swap):**
1. Agent calls tool with params, `confirmed: false` (or omitted)
2. Tool runs command function in preview/dry-run mode, returns preview (amounts, fees, recipient)
3. Agent presents preview to user, gets confirmation
4. Agent calls tool again with same params + `confirmed: true`
5. Tool executes transaction, returns tx hash + explorer URL

Maps directly to CLI's existing `--dry-run` logic — same code path, toggled by `confirmed` instead of CLI flag.

**Per-call data flow:**
```
MCP JSON-RPC request
  → tools.ts handler: validate input, map to command fn args
    → command fn: calls withVault(), returns typed result
  → tools.ts handler: format result as MCP tool response
MCP JSON-RPC response
```

## Server Lifecycle & Error Handling

**Startup:** Commander subcommand creates MCP Server with stdio transport, registers tools, blocks on stdin.

**Vault init is per tool call, not at startup.** Benefits:
- Instant server start (no upfront WASM loading)
- Clear error on first tool call if credentials missing
- Picks up new credentials if user runs `vasig auth` after server start

**Error mapping (reuses lib/errors.ts classification):**

| Error type | MCP behavior |
|-----------|-------------|
| Auth missing | Tool error: "Vault not authenticated. Run `vasig auth` first." |
| Network/timeout | Tool error with retry suggestion |
| Invalid input | Tool error with suggestion (reuses fuzzy match from validation.ts) |
| Signing timeout | Tool error noting MPC peer may be offline |
| SDK/unexpected | Tool error with cleaned message (reuses error scrubbing) |

All logging to stderr. MCP SDK owns stdout for JSON-RPC exclusively.

## Testing Approach

**Unit tests** for `src/mcp/tools.ts`:
- Input validation (missing/invalid params → MCP errors)
- Correct command function called with correct args
- Confirmation flow: `confirmed: false` → preview, `confirmed: true` → execute
- Error classification → clear MCP error messages

**Integration test** — MCP server as subprocess, JSON-RPC over stdio:
- Server responds to `tools/list`
- Read-only tools return structured data
- send/swap without `confirmed` return previews

No re-testing of SDK/signing logic from MCP side — that's the command functions' responsibility.

## Approved Approach

Approach 1: MCP as thin adapter over existing command functions. Single package, `vasig mcp` subcommand, single active vault, `confirmed` flag for destructive operations, vault init per tool call. Maximal code sharing with CLI as first-class citizen.

**2026-03-23T02:30:00Z**

# Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use /run tk:v-ugin to implement this plan task-by-task via subagent-driven-development.

**Goal:** Create an MCP server (`vasig mcp`) that exposes Vultisig vault operations as tools for AI agents like Claude Code.

**Architecture:** Thin MCP adapter layer over existing CLI command functions. Two new files (`src/mcp/index.ts`, `src/mcp/tools.ts`) wrap `getBalances`, `executeSend`, `getSwapQuote`, `executeSwap`, `getAddresses`, `getVaultInfo`, `getSupportedChains` with JSON Schema tool definitions and MCP response formatting. Vault init is per tool call (lazy), not at startup.

**Tech Stack:** `@modelcontextprotocol/sdk` for MCP server/stdio transport, existing `@vultisig/sdk` via shared command functions, vitest for testing.

---

### Task 1: Refactor executeSwap to not take format parameter

The `executeSwap` function currently takes `format: OutputFormat` and calls `printResult` mid-execution (line 122-124 of `src/commands/swap.ts`). This mixes I/O into the data function. Move the mid-execution print into `swapCommand` so `executeSwap` returns data cleanly.

**Files:**
- Modify: `src/commands/swap.ts:104-177`
- Test: `tests/commands/swap.test.ts`

**Step 1: Update executeSwap signature — remove format param**

In `src/commands/swap.ts`, change `executeSwap` to not take `format` and not call `printResult`:

```typescript
export async function executeSwap(opts: SwapOpts, vaultId?: string): Promise<SwapResult | SwapDryRunResult> {
  return withVault(async ({ vault }) => {
    const { from, to, quoteRequest } = buildSwapQuoteParams(opts)
    const quote = await vault.getSwapQuote(quoteRequest)

    if ((quote.estimatedOutputFiat == null || quote.estimatedOutputFiat < 0.01) && parseFloat(opts.amount) > 0) {
      throw new NoRouteError(
        'Quote output is near-zero — this route would result in fund loss. Try a different route.',
        'Try a different token pair or a larger amount',
      )
    }

    const summary = mapQuoteResult(quote, opts.amount)

    if (opts.dryRun) {
      return { dryRun: true as const, ...summary }
    }

    const { keysignPayload, approvalPayload } = await vault.prepareSwapTx({
      fromCoin: { chain: from.chain, token: from.token },
      toCoin: { chain: to.chain, token: to.token },
      amount: parseFloat(opts.amount),
      swapQuote: quote,
      autoApprove: false,
    })

    const validation = await vault.validateTransaction(keysignPayload)
    if (validation?.isRisky && !opts.yes) {
      throw new UsageError(
        `Swap flagged as risky (${validation.riskLevel}): ${validation.description}`,
        `Details: ${validation.features.join(', ')}. Use --yes to override.`
      )
    }
    if (validation?.isRisky) {
      process.stderr.write(`⚠ Risk warning (${validation.riskLevel}): ${validation.description}\n`)
    }

    let approvalTxHash: string | undefined

    if (approvalPayload) {
      const approvalHashes = await vault.extractMessageHashes(approvalPayload)
      const approvalSig = await signWithRetry(() =>
        vault.sign({ transaction: approvalPayload, chain: from.chain, messageHashes: approvalHashes }),
      )
      approvalTxHash = await vault.broadcastTx({
        chain: from.chain,
        keysignPayload: approvalPayload,
        signature: approvalSig,
      })
      await new Promise((resolve) => setTimeout(resolve, 5000))
    }

    const messageHashes = await vault.extractMessageHashes(keysignPayload)
    const signature = await signWithRetry(() =>
      vault.sign({ transaction: keysignPayload, chain: from.chain, messageHashes }),
    )

    const txHash = await vault.broadcastTx({
      chain: from.chain,
      keysignPayload,
      signature,
    })

    const explorerUrl = Vultisig.getTxExplorerUrl(from.chain, txHash)

    const result: SwapResult = { txHash, chain: from.chain, explorerUrl }
    if (approvalTxHash) result.approvalTxHash = approvalTxHash
    return result as SwapResult
  }, vaultId)
}
```

Remove the `import { printResult } from '../lib/output.js'` line (and `import type { OutputFormat } from '../lib/output.js'` if only used by executeSwap — check swapCommand/swapQuoteCommand/swapChainsCommand first; they use it so keep the import).

**Step 2: Update swapCommand to print quote before executing**

```typescript
export async function swapCommand(opts: SwapOpts, format: OutputFormat, vaultId?: string): Promise<void> {
  if (!opts.dryRun && format !== 'json') {
    const quote = await getSwapQuote(opts, vaultId)
    printResult({ action: 'quote', ...quote }, format)
  }
  const result = await executeSwap(opts, vaultId)
  printResult(result, format)
}
```

Note: This changes behavior slightly — the quote is now fetched separately before execution instead of mid-execution. This means two vault loads instead of one for table-format swaps. That's acceptable for the CLI path since it only affects the visual preview.

Actually, simpler approach — just remove the mid-execution print from `executeSwap` and don't add it to `swapCommand`. The swap result already contains all the info. The mid-execution print was a nice-to-have that showed the quote before signing, but the final result is what matters.

Simpler fix:

```typescript
export async function executeSwap(opts: SwapOpts, vaultId?: string): Promise<SwapResult | SwapDryRunResult> {
```

Just remove the `format` param and the `if (format !== 'json')` block (lines 122-124). Keep everything else the same.

**Step 3: Update swapCommand call site**

```typescript
export async function swapCommand(opts: SwapOpts, format: OutputFormat, vaultId?: string): Promise<void> {
  const result = await executeSwap(opts, vaultId)
  printResult(result, format)
}
```

**Step 4: Update the call site in index.ts**

In `src/index.ts`, the swap execute action calls `swapCommand` which now calls `executeSwap` without format. No change needed in `index.ts` since `swapCommand` still takes `format`.

**Step 5: Run existing tests to verify nothing broke**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/commands/swap.test.ts`

The test for `executeSwap` passes `yes: true` without a format param in assertions, so the test should still pass after updating the mock call. Actually, the test already calls `executeSwap` without a format param — wait, no, checking the test:

```typescript
const result = await executeSwap({
  from: 'Ethereum',
  to: 'Bitcoin',
  amount: '1.0',
  yes: true,
})
```

This doesn't pass `format` currently. But the actual function signature requires it. Wait — the test file imports `executeSwap` and the current signature is `(opts, format, vaultId?)`. The test only passes `opts`. TypeScript would catch this... unless the mock intercepts. Let me check — the `withVault` mock makes it work because the function body runs inside `withVault`.

After removing the `format` param, the test call matches the new signature perfectly. No test changes needed.

**Step 6: Run all tests**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run`
Expected: All tests pass.

**Step 7: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/commands/swap.ts
git commit -m "refactor: remove format param from executeSwap for MCP reuse"
```

---

### Task 2: Install @modelcontextprotocol/sdk dependency

**Files:**
- Modify: `package.json`

**Step 1: Install the MCP SDK**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npm install @modelcontextprotocol/sdk`

**Step 2: Verify installation**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && node -e "import('@modelcontextprotocol/sdk').then(m => console.log('OK', Object.keys(m).slice(0,5)))"`
Expected: Prints 'OK' with some exported keys.

**Step 3: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add package.json package-lock.json
git commit -m "deps: add @modelcontextprotocol/sdk for MCP server"
```

---

### Task 3: Create MCP tools module with read-only tools

**Files:**
- Create: `src/mcp/tools.ts`
- Test: `tests/mcp/tools.test.ts`

**Step 1: Write the failing test for tool registration**

Create `tests/mcp/tools.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'

const mockVault = vi.hoisted(() => ({
  id: 'vault-123',
  name: 'TestVault',
  type: '2of3',
  isEncrypted: false,
  threshold: 2,
  totalSigners: 3,
  chains: ['Ethereum', 'Bitcoin'],
  balance: vi.fn(),
  balances: vi.fn(),
  balancesWithPrices: vi.fn(),
  address: vi.fn(),
  getTokens: vi.fn().mockReturnValue([]),
  discoverTokens: vi.fn().mockResolvedValue([]),
  addToken: vi.fn(),
  getSwapQuote: vi.fn(),
  getSupportedSwapChains: vi.fn(),
  prepareSendTx: vi.fn(),
  prepareSwapTx: vi.fn(),
  validateTransaction: vi.fn().mockResolvedValue(null),
  extractMessageHashes: vi.fn().mockResolvedValue(['0xhash']),
  sign: vi.fn().mockResolvedValue({ signature: '0xsig' }),
  broadcastTx: vi.fn().mockResolvedValue('0xtxhash'),
  getMaxSendAmount: vi.fn(),
  on: vi.fn(),
  removeAllListeners: vi.fn(),
  isUnlocked: vi.fn().mockReturnValue(true),
}))

vi.mock('../../src/lib/sdk.js', () => ({
  withVault: vi.fn(async (fn) => fn({
    sdk: { dispose: vi.fn() },
    vault: mockVault,
    vaultEntry: { id: 'vault-123', name: 'TestVault', filePath: '/test.vult', tokens: {} },
  })),
  suppressConsoleWarn: vi.fn(async (fn: () => unknown) => fn()),
}))

vi.mock('../../src/auth/config.js', () => ({
  loadConfig: vi.fn().mockResolvedValue({
    vaults: [{ id: 'vault-123', name: 'TestVault', filePath: '/test.vult' }],
  }),
  persistTokens: vi.fn(),
}))

vi.mock('../../src/lib/tokens.js', () => ({
  discoverAndPersistTokens: vi.fn().mockResolvedValue([]),
}))

vi.mock('@vultisig/sdk', () => ({
  Vultisig: { getTxExplorerUrl: vi.fn().mockReturnValue('https://etherscan.io/tx/0xtxhash') },
  SUPPORTED_CHAINS: ['Ethereum', 'Bitcoin', 'THORChain', 'Solana'],
}))

vi.mock('keytar', () => ({
  default: {
    getPassword: vi.fn().mockResolvedValue('server-pw'),
    setPassword: vi.fn(),
    deletePassword: vi.fn(),
  },
}))

import { getTools } from '../../src/mcp/tools.js'

describe('MCP tools', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockVault.chains = ['Ethereum', 'Bitcoin']
  })

  it('registers all 8 tools', () => {
    const tools = getTools()
    expect(Object.keys(tools)).toHaveLength(8)
    expect(Object.keys(tools)).toEqual(
      expect.arrayContaining([
        'get_balances', 'get_portfolio', 'get_address',
        'vault_info', 'supported_chains', 'swap_quote',
        'send', 'swap',
      ])
    )
  })

  it('each tool has description and inputSchema', () => {
    const tools = getTools()
    for (const [name, tool] of Object.entries(tools)) {
      expect(tool.description, `${name} missing description`).toBeTruthy()
      expect(tool.inputSchema, `${name} missing inputSchema`).toBeTruthy()
      expect(tool.inputSchema.type).toBe('object')
    }
  })

  describe('get_balances', () => {
    it('returns balances for all chains', async () => {
      mockVault.balances.mockResolvedValue({
        Ethereum: { chainId: 'Ethereum', symbol: 'ETH', amount: '1000000000000000000', formattedAmount: '1.0', decimals: 18 },
      })

      const tools = getTools()
      const result = await tools.get_balances.handler({})
      expect(result.content[0].type).toBe('text')
      const data = JSON.parse(result.content[0].text)
      expect(data[0].chain).toBe('Ethereum')
      expect(data[0].symbol).toBe('ETH')
    })

    it('passes chain filter when provided', async () => {
      mockVault.balance.mockResolvedValue({
        chainId: 'Ethereum', symbol: 'ETH', amount: '1000000000000000000', formattedAmount: '1.0', decimals: 18,
      })

      const tools = getTools()
      await tools.get_balances.handler({ chain: 'Ethereum' })
      expect(mockVault.balance).toHaveBeenCalled()
    })
  })

  describe('get_portfolio', () => {
    it('returns balances with fiat values', async () => {
      mockVault.balancesWithPrices.mockResolvedValue({
        Ethereum: { chainId: 'Ethereum', symbol: 'ETH', amount: '1e18', formattedAmount: '1.0', decimals: 18, fiatValue: 2500.0 },
      })

      const tools = getTools()
      const result = await tools.get_portfolio.handler({})
      const data = JSON.parse(result.content[0].text)
      expect(data[0].fiatCurrency).toBe('USD')
    })
  })

  describe('get_address', () => {
    it('returns address for a chain', async () => {
      mockVault.address.mockResolvedValue('0xAbC123')

      const tools = getTools()
      const result = await tools.get_address.handler({ chain: 'Ethereum' })
      const data = JSON.parse(result.content[0].text)
      expect(data.chain).toBe('Ethereum')
      expect(data.address).toBe('0xAbC123')
    })
  })

  describe('vault_info', () => {
    it('returns vault metadata', async () => {
      const tools = getTools()
      const result = await tools.vault_info.handler({})
      const data = JSON.parse(result.content[0].text)
      expect(data.name).toBe('TestVault')
      expect(data.chains).toEqual(['Ethereum', 'Bitcoin'])
    })
  })

  describe('supported_chains', () => {
    it('returns supported swap chains', async () => {
      mockVault.getSupportedSwapChains.mockResolvedValue(['Ethereum', 'Bitcoin', 'THORChain'])

      const tools = getTools()
      const result = await tools.supported_chains.handler({})
      const data = JSON.parse(result.content[0].text)
      expect(data).toEqual(['Ethereum', 'Bitcoin', 'THORChain'])
    })
  })

  describe('swap_quote', () => {
    it('returns a quote', async () => {
      mockVault.getSwapQuote.mockResolvedValue({
        fromCoin: { chain: 'Ethereum', ticker: 'ETH', decimals: 18 },
        toCoin: { chain: 'Bitcoin', ticker: 'BTC', decimals: 8 },
        estimatedOutput: 5000000n,
        estimatedOutputFiat: 2500.0,
        provider: 'thorchain',
        warnings: [],
      })

      const tools = getTools()
      const result = await tools.swap_quote.handler({
        from: 'Ethereum',
        to: 'Bitcoin',
        amount: '1.0',
      })
      const data = JSON.parse(result.content[0].text)
      expect(data.fromToken).toBe('ETH')
      expect(data.estimatedOutput).toBe('0.05')
    })
  })
})
```

**Step 2: Run the test to verify it fails**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/tools.test.ts`
Expected: FAIL — cannot find module `../../src/mcp/tools.js`

**Step 3: Write the tools module**

Create `src/mcp/tools.ts`:

```typescript
import { getBalances } from '../commands/balance.js'
import { getAddresses } from '../commands/addresses.js'
import { getVaultInfo } from '../commands/vault.js'
import { getSwapQuote, getSupportedChains, executeSwap } from '../commands/swap.js'
import { executeSend } from '../commands/send.js'
import { VasigError, classifyError } from '../lib/errors.js'

interface ToolInput {
  [key: string]: unknown
}

interface ToolResult {
  content: Array<{ type: 'text'; text: string }>
  isError?: boolean
}

interface ToolDef {
  description: string
  inputSchema: {
    type: 'object'
    properties: Record<string, unknown>
    required?: string[]
  }
  handler: (input: ToolInput) => Promise<ToolResult>
}

function success(data: unknown): ToolResult {
  return { content: [{ type: 'text', text: JSON.stringify(data, null, 2) }] }
}

function error(message: string, hint?: string): ToolResult {
  const text = hint ? `${message}\n\nHint: ${hint}` : message
  return { content: [{ type: 'text', text }], isError: true }
}

async function wrapHandler<T>(fn: () => Promise<T>): Promise<ToolResult> {
  try {
    const data = await fn()
    return success(data)
  } catch (err: unknown) {
    if (err instanceof VasigError) {
      return error(err.message, err.hint)
    }
    if (err instanceof Error) {
      const classified = classifyError(err)
      return error(classified.message, classified.hint)
    }
    return error(String(err))
  }
}

export function getTools(): Record<string, ToolDef> {
  return {
    get_balances: {
      description: 'Get token balances for a specific chain or all chains on the vault',
      inputSchema: {
        type: 'object',
        properties: {
          chain: { type: 'string', description: 'Filter by chain (e.g. "Ethereum", "Bitcoin"). Omit for all chains.' },
          include_tokens: { type: 'boolean', description: 'Include ERC-20/SPL token balances (default: false)' },
        },
      },
      handler: (input) => wrapHandler(() =>
        getBalances({ chain: input.chain as string | undefined, includeTokens: input.include_tokens as boolean | undefined }),
      ),
    },

    get_portfolio: {
      description: 'Get token balances with USD fiat values for portfolio overview',
      inputSchema: {
        type: 'object',
        properties: {
          chain: { type: 'string', description: 'Filter by chain. Omit for all chains.' },
          include_tokens: { type: 'boolean', description: 'Include ERC-20/SPL token balances (default: false)' },
        },
      },
      handler: (input) => wrapHandler(() =>
        getBalances({ chain: input.chain as string | undefined, includeTokens: input.include_tokens as boolean | undefined, fiat: true }),
      ),
    },

    get_address: {
      description: 'Get the derived wallet address for a specific chain',
      inputSchema: {
        type: 'object',
        properties: {
          chain: { type: 'string', description: 'Chain to get address for (e.g. "Ethereum", "Bitcoin")' },
        },
        required: ['chain'],
      },
      handler: (input) => wrapHandler(async () => {
        const results = await getAddresses()
        const chain = (input.chain as string).toLowerCase()
        const match = results.find((r) => r.chain.toLowerCase() === chain)
        if (match) return match
        return results
      }),
    },

    vault_info: {
      description: 'Show vault name, type, active chains, signer threshold, and encryption status',
      inputSchema: {
        type: 'object',
        properties: {},
      },
      handler: () => wrapHandler(() => getVaultInfo()),
    },

    supported_chains: {
      description: 'List all chains supported for cross-chain swaps',
      inputSchema: {
        type: 'object',
        properties: {},
      },
      handler: () => wrapHandler(() => getSupportedChains()),
    },

    swap_quote: {
      description: 'Get a swap quote showing estimated output, fees, and provider without executing. Use chain:token format (e.g. "Ethereum:USDC") for tokens, or just chain name for native coins.',
      inputSchema: {
        type: 'object',
        properties: {
          from: { type: 'string', description: 'Source chain or chain:token (e.g. "Ethereum" or "Ethereum:USDC")' },
          to: { type: 'string', description: 'Destination chain or chain:token (e.g. "Bitcoin" or "Arbitrum:USDC")' },
          amount: { type: 'string', description: 'Amount to swap (e.g. "1.5")' },
        },
        required: ['from', 'to', 'amount'],
      },
      handler: (input) => wrapHandler(() =>
        getSwapQuote({ from: input.from as string, to: input.to as string, amount: input.amount as string }),
      ),
    },

    send: {
      description: 'Send tokens to an address. First call with confirmed=false (or omit) to preview the transaction. Then call again with confirmed=true to execute it.',
      inputSchema: {
        type: 'object',
        properties: {
          chain: { type: 'string', description: 'Blockchain (e.g. "Ethereum", "Bitcoin")' },
          to: { type: 'string', description: 'Recipient address' },
          amount: { type: 'string', description: 'Amount to send (e.g. "0.1") or "max"' },
          token: { type: 'string', description: 'Token contract address (omit for native coin)' },
          memo: { type: 'string', description: 'Transaction memo (non-EVM chains only)' },
          confirmed: { type: 'boolean', description: 'false=preview only, true=execute transaction. Always preview first and show the user before confirming.' },
        },
        required: ['chain', 'to', 'amount'],
      },
      handler: (input) => wrapHandler(() =>
        executeSend({
          chain: input.chain as string,
          to: input.to as string,
          amount: input.amount as string,
          token: input.token as string | undefined,
          memo: input.memo as string | undefined,
          dryRun: !(input.confirmed as boolean),
          yes: true,
        }),
      ),
    },

    swap: {
      description: 'Swap tokens between chains. First call with confirmed=false (or omit) to preview the swap quote. Then call again with confirmed=true to execute it. Use chain:token format for tokens.',
      inputSchema: {
        type: 'object',
        properties: {
          from: { type: 'string', description: 'Source chain or chain:token (e.g. "Ethereum" or "Ethereum:USDC")' },
          to: { type: 'string', description: 'Destination chain or chain:token (e.g. "Bitcoin" or "Arbitrum:USDC")' },
          amount: { type: 'string', description: 'Amount to swap (e.g. "1.5")' },
          confirmed: { type: 'boolean', description: 'false=preview only (returns quote), true=execute swap. Always preview first and show the user before confirming.' },
        },
        required: ['from', 'to', 'amount'],
      },
      handler: (input) => wrapHandler(() =>
        executeSwap({
          from: input.from as string,
          to: input.to as string,
          amount: input.amount as string,
          dryRun: !(input.confirmed as boolean),
          yes: true,
        }),
      ),
    },
  }
}
```

**Step 4: Run tests to verify they pass**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/tools.test.ts`
Expected: All tests pass.

**Step 5: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/mcp/tools.ts tests/mcp/tools.test.ts
git commit -m "feat: add MCP tool definitions wrapping existing command functions"
```

---

### Task 4: Write tests for send/swap confirmation flow in MCP tools

**Files:**
- Modify: `tests/mcp/tools.test.ts`

**Step 1: Add send confirmation tests**

Append to `tests/mcp/tools.test.ts`, inside the `describe('MCP tools')` block:

```typescript
  describe('send', () => {
    it('returns preview when confirmed is false', async () => {
      mockVault.address.mockResolvedValue('0xSender')
      mockVault.balance.mockResolvedValue({
        decimals: 18, symbol: 'ETH', chain: 'Ethereum',
        amount: '10000000000000000000', formattedAmount: '10.0',
      })

      const tools = getTools()
      const result = await tools.send.handler({
        chain: 'Ethereum',
        to: '0xRecipient',
        amount: '1.0',
      })

      const data = JSON.parse(result.content[0].text)
      expect(data.dryRun).toBe(true)
      expect(data.chain).toBe('Ethereum')
      expect(data.to).toBe('0xRecipient')
      expect(data.amount).toBe('1.0')
      expect(result.isError).toBeUndefined()
      expect(mockVault.sign).not.toHaveBeenCalled()
    })

    it('executes transaction when confirmed is true', async () => {
      mockVault.address.mockResolvedValue('0xSender')
      mockVault.balance.mockResolvedValue({
        decimals: 18, symbol: 'ETH', chain: 'Ethereum',
        amount: '10000000000000000000', formattedAmount: '10.0',
      })
      mockVault.prepareSendTx.mockResolvedValue({ coin: { chain: 'Ethereum' } })

      const tools = getTools()
      const result = await tools.send.handler({
        chain: 'Ethereum',
        to: '0xRecipient',
        amount: '1.0',
        confirmed: true,
      })

      const data = JSON.parse(result.content[0].text)
      expect(data.txHash).toBe('0xtxhash')
      expect(data.explorerUrl).toBeTruthy()
      expect(result.isError).toBeUndefined()
      expect(mockVault.sign).toHaveBeenCalled()
    })

    it('returns error for insufficient balance on confirmed send', async () => {
      mockVault.address.mockResolvedValue('0xSender')
      mockVault.balance.mockResolvedValue({
        decimals: 18, symbol: 'ETH', chain: 'Ethereum',
        amount: '500000000000000000', formattedAmount: '0.5',
      })

      const tools = getTools()
      const result = await tools.send.handler({
        chain: 'Ethereum',
        to: '0xRecipient',
        amount: '1.0',
        confirmed: true,
      })

      expect(result.isError).toBe(true)
      expect(result.content[0].text).toContain('Insufficient balance')
    })
  })

  describe('swap', () => {
    it('returns quote when confirmed is false', async () => {
      mockVault.getSwapQuote.mockResolvedValue({
        fromCoin: { chain: 'Ethereum', ticker: 'ETH', decimals: 18 },
        toCoin: { chain: 'Bitcoin', ticker: 'BTC', decimals: 8 },
        estimatedOutput: 5000000n,
        estimatedOutputFiat: 2500.0,
        provider: 'thorchain',
        warnings: [],
      })

      const tools = getTools()
      const result = await tools.swap.handler({
        from: 'Ethereum',
        to: 'Bitcoin',
        amount: '1.0',
      })

      const data = JSON.parse(result.content[0].text)
      expect(data.dryRun).toBe(true)
      expect(data.fromToken).toBe('ETH')
      expect(data.toToken).toBe('BTC')
      expect(result.isError).toBeUndefined()
      expect(mockVault.sign).not.toHaveBeenCalled()
    })

    it('executes swap when confirmed is true', async () => {
      mockVault.getSwapQuote.mockResolvedValue({
        fromCoin: { chain: 'Ethereum', ticker: 'ETH', decimals: 18 },
        toCoin: { chain: 'Bitcoin', ticker: 'BTC', decimals: 8 },
        estimatedOutput: 5000000n,
        estimatedOutputFiat: 2500.0,
        provider: 'thorchain',
        warnings: [],
      })
      mockVault.prepareSwapTx.mockResolvedValue({
        keysignPayload: { coin: { chain: 'Ethereum' } },
        approvalPayload: null,
      })

      const tools = getTools()
      const result = await tools.swap.handler({
        from: 'Ethereum',
        to: 'Bitcoin',
        amount: '1.0',
        confirmed: true,
      })

      const data = JSON.parse(result.content[0].text)
      expect(data.txHash).toBeTruthy()
      expect(result.isError).toBeUndefined()
      expect(mockVault.sign).toHaveBeenCalled()
    })

    it('returns error for invalid swap params', async () => {
      const tools = getTools()
      const result = await tools.swap.handler({
        from: 'Ethereum',
        to: 'Ethereum',
        amount: '1.0',
        confirmed: true,
      })

      expect(result.isError).toBe(true)
      expect(result.content[0].text).toContain('Cannot swap the same token')
    })
  })
```

**Step 2: Run the tests**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/tools.test.ts`
Expected: All tests pass (the handler code was written in Task 3).

**Step 3: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add tests/mcp/tools.test.ts
git commit -m "test: add MCP send/swap confirmation flow tests"
```

---

### Task 5: Create MCP server entry point

**Files:**
- Create: `src/mcp/index.ts`
- Test: `tests/mcp/server.test.ts`

**Step 1: Write the failing test for server creation**

Create `tests/mcp/server.test.ts`:

```typescript
import { describe, it, expect, vi } from 'vitest'

vi.mock('@modelcontextprotocol/sdk/server/index.js', () => ({
  McpServer: vi.fn().mockImplementation(() => ({
    tool: vi.fn(),
    connect: vi.fn(),
  })),
}))

vi.mock('@modelcontextprotocol/sdk/server/stdio.js', () => ({
  StdioServerTransport: vi.fn().mockImplementation(() => ({})),
}))

// Mock all command modules to avoid SDK initialization
vi.mock('../../src/commands/balance.js', () => ({ getBalances: vi.fn() }))
vi.mock('../../src/commands/addresses.js', () => ({ getAddresses: vi.fn() }))
vi.mock('../../src/commands/vault.js', () => ({ getVaultInfo: vi.fn() }))
vi.mock('../../src/commands/swap.js', () => ({
  getSwapQuote: vi.fn(),
  getSupportedChains: vi.fn(),
  executeSwap: vi.fn(),
}))
vi.mock('../../src/commands/send.js', () => ({ executeSend: vi.fn() }))
vi.mock('../../src/lib/errors.js', () => ({
  VasigError: class VasigError extends Error {},
  classifyError: vi.fn((err: Error) => err),
}))

import { createMcpServer } from '../../src/mcp/index.js'
import { McpServer } from '@modelcontextprotocol/sdk/server/index.js'

describe('MCP server', () => {
  it('creates a server with correct name and version', () => {
    const server = createMcpServer()
    expect(McpServer).toHaveBeenCalledWith(
      expect.objectContaining({
        name: 'vultisig',
      }),
    )
    expect(server).toBeDefined()
  })

  it('registers all 8 tools', () => {
    const server = createMcpServer()
    const mockInstance = (McpServer as any).mock.results[0].value
    expect(mockInstance.tool).toHaveBeenCalledTimes(8)
  })

  it('registers tools with expected names', () => {
    const server = createMcpServer()
    const mockInstance = (McpServer as any).mock.results[0].value
    const toolNames = mockInstance.tool.mock.calls.map((c: any[]) => c[0])
    expect(toolNames).toEqual(expect.arrayContaining([
      'get_balances', 'get_portfolio', 'get_address',
      'vault_info', 'supported_chains', 'swap_quote',
      'send', 'swap',
    ]))
  })
})
```

**Step 2: Run the test to verify it fails**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/server.test.ts`
Expected: FAIL — cannot find module `../../src/mcp/index.js`

**Step 3: Write the MCP server entry point**

Create `src/mcp/index.ts`:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/index.js'
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js'
import { getTools } from './tools.js'

export function createMcpServer(): McpServer {
  const server = new McpServer({
    name: 'vultisig',
    version: '0.1.0',
  })

  const tools = getTools()

  for (const [name, tool] of Object.entries(tools)) {
    server.tool(
      name,
      tool.description,
      tool.inputSchema.properties,
      async (params) => tool.handler(params),
    )
  }

  return server
}

export async function startMcpServer(): Promise<void> {
  const server = createMcpServer()
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

**Step 4: Run the test to verify it passes**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/server.test.ts`
Expected: All tests pass.

**Step 5: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/mcp/index.ts tests/mcp/server.test.ts
git commit -m "feat: add MCP server entry point with stdio transport"
```

---

### Task 6: Add `vasig mcp` subcommand to CLI

**Files:**
- Modify: `src/index.ts`

**Step 1: Write failing test for the mcp command**

This is best tested as an integration test. Add to `tests/mcp/server.test.ts`:

```typescript
describe('vasig mcp command registration', () => {
  it('mcp command is registered in CLI', async () => {
    // We verify by checking that the command exists in the program
    // This is a lightweight check — full integration test is in Task 8
    const { getCommandFromArgv } = await import('../../src/index.js')
    expect(getCommandFromArgv(['mcp'])).toBe('mcp')
  })
})
```

Actually, importing `index.ts` will execute `main()` which calls `program.parseAsync()`. That's not testable in a unit test. Skip this approach — add the command directly and test via the integration test in Task 8.

**Step 1: Add mcp command to index.ts**

Add the following block to `src/index.ts`, after the `schema` command block (around line 331) and before the `getFormat()` function:

```typescript
program
  .command('mcp')
  .description('[Integration] Start MCP server for AI agent integration')
  .addHelpText('after', `
The MCP server exposes vault operations as tools for AI agents (e.g. Claude Code).
It communicates via JSON-RPC over stdin/stdout (stdio transport).

Setup:
  claude mcp add vultisig -- vasig mcp

Tools provided:
  get_balances      Get token balances
  get_portfolio     Get balances with USD values
  get_address       Get wallet address for a chain
  vault_info        Show vault details
  supported_chains  List swap-supported chains
  swap_quote        Get a swap quote
  send              Send tokens (with confirmation)
  swap              Swap tokens (with confirmation)`)
  .action(async () => {
    const { startMcpServer } = await import('./mcp/index.js')
    await startMcpServer()
  })
```

**Step 2: Verify the command is registered**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx tsx src/index.ts mcp --help`
Expected: Shows the mcp command help text with tool list.

**Step 3: Run all tests to verify nothing broke**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run`
Expected: All tests pass.

**Step 4: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/index.ts
git commit -m "feat: add 'vasig mcp' subcommand for AI agent integration"
```

---

### Task 7: Suppress stdout pollution in MCP mode

The MCP server must ensure no `console.log` calls leak to stdout. The SDK and some libraries use `console.log` during operations (e.g., signing). The `signWithRetry` function already suppresses `console.log`, but we should add a global safety net for MCP mode.

**Files:**
- Modify: `src/mcp/index.ts`

**Step 1: Add console.log suppression to startMcpServer**

In `src/mcp/index.ts`, modify `startMcpServer`:

```typescript
export async function startMcpServer(): Promise<void> {
  // MCP stdio requires stdout exclusively for JSON-RPC.
  // Redirect any console.log to stderr to prevent protocol corruption.
  console.log = (...args: unknown[]) => {
    console.error('[mcp:log]', ...args)
  }

  const server = createMcpServer()
  const transport = new StdioServerTransport()
  await server.connect(transport)
}
```

**Step 2: Run all tests**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run`
Expected: All tests pass.

**Step 3: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/mcp/index.ts
git commit -m "fix: redirect console.log to stderr in MCP mode to protect stdio"
```

---

### Task 8: Integration test — MCP server over stdio

**Files:**
- Create: `tests/mcp/integration.test.ts`

**Step 1: Write the integration test**

Create `tests/mcp/integration.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { spawn } from 'node:child_process'
import { join } from 'node:path'

function sendJsonRpc(proc: ReturnType<typeof spawn>, method: string, params?: unknown, id = 1): void {
  const msg = JSON.stringify({ jsonrpc: '2.0', id, method, params })
  const header = `Content-Length: ${Buffer.byteLength(msg)}\r\n\r\n`
  proc.stdin!.write(header + msg)
}

function readResponse(proc: ReturnType<typeof spawn>, timeoutMs = 10000): Promise<any> {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error('Timeout waiting for MCP response')), timeoutMs)
    let buffer = ''
    const onData = (chunk: Buffer) => {
      buffer += chunk.toString()
      // MCP uses Content-Length framing
      const headerEnd = buffer.indexOf('\r\n\r\n')
      if (headerEnd === -1) return
      const header = buffer.slice(0, headerEnd)
      const lengthMatch = header.match(/Content-Length:\s*(\d+)/i)
      if (!lengthMatch) return
      const contentLength = parseInt(lengthMatch[1], 10)
      const bodyStart = headerEnd + 4
      if (buffer.length < bodyStart + contentLength) return
      const body = buffer.slice(bodyStart, bodyStart + contentLength)
      clearTimeout(timeout)
      proc.stdout!.off('data', onData)
      resolve(JSON.parse(body))
    }
    proc.stdout!.on('data', onData)
  })
}

describe('MCP server integration', () => {
  it('responds to initialize and tools/list', async () => {
    const proc = spawn('npx', ['tsx', join(__dirname, '../../src/mcp/start.ts')], {
      stdio: ['pipe', 'pipe', 'pipe'],
      env: { ...process.env, NODE_NO_WARNINGS: '1' },
    })

    try {
      // Send initialize
      sendJsonRpc(proc, 'initialize', {
        protocolVersion: '2024-11-05',
        capabilities: {},
        clientInfo: { name: 'test', version: '0.0.1' },
      })
      const initResp = await readResponse(proc)
      expect(initResp.result.serverInfo.name).toBe('vultisig')

      // Send initialized notification (no id)
      const notif = JSON.stringify({ jsonrpc: '2.0', method: 'notifications/initialized' })
      proc.stdin!.write(`Content-Length: ${Buffer.byteLength(notif)}\r\n\r\n${notif}`)

      // List tools
      sendJsonRpc(proc, 'tools/list', {}, 2)
      const listResp = await readResponse(proc)
      const toolNames = listResp.result.tools.map((t: any) => t.name)
      expect(toolNames).toHaveLength(8)
      expect(toolNames).toContain('get_balances')
      expect(toolNames).toContain('send')
      expect(toolNames).toContain('swap')
    } finally {
      proc.kill()
    }
  }, 15000)
})
```

**Step 2: Create the MCP start script**

Create `src/mcp/start.ts` (a minimal entry point for the integration test and for `npx` usage):

```typescript
import { startMcpServer } from './index.js'

startMcpServer().catch((err) => {
  console.error('MCP server failed to start:', err)
  process.exit(1)
})
```

**Step 3: Run the integration test**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run tests/mcp/integration.test.ts`
Expected: Test passes — server starts, responds to initialize, lists 8 tools.

**Step 4: Run all tests**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run`
Expected: All tests pass.

**Step 5: Commit**

```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add src/mcp/start.ts tests/mcp/integration.test.ts
git commit -m "test: add MCP server integration test over stdio"
```

---

### Task 9: Lint and type-check

**Files:**
- All new/modified files

**Step 1: Run TypeScript type-check**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx tsc --noEmit`
Expected: No errors. If there are type errors, fix them.

**Step 2: Run all tests one final time**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx vitest run`
Expected: All tests pass.

**Step 3: Commit any fixes**

If any fixes were needed:
```bash
cd /home/sean/Repos/vultisig/vultiagent-cli
git add -A
git commit -m "fix: resolve type errors in MCP server"
```

---

### Task 10: Verify Claude Code MCP setup works

**Step 1: Test the Claude Code install command**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && npx tsx src/index.ts mcp --help`
Expected: Help output shows the setup instructions.

**Step 2: Verify the server starts and shuts down cleanly**

Run: `cd /home/sean/Repos/vultisig/vultiagent-cli && echo '{}' | timeout 2 npx tsx src/mcp/start.ts 2>/dev/null; echo "exit: $?"`
Expected: Server starts (and times out after 2s since no valid JSON-RPC was sent). Non-zero exit from timeout is fine.

**Step 3: Final commit — update package.json bin entry (optional)**

If we want `vasig mcp` to also work as `npx @vultisig/mcp`:
- This requires publishing as a separate package or adding a `bin` entry. Skip for now — `vasig mcp` is the primary entry point, and `claude mcp add vultisig -- vasig mcp` is the documented setup.

No commit needed if no changes.

**2026-03-23T03:05:27Z**

Tasks 1-8 complete: refactored executeSwap, installed MCP SDK, created tools module (Zod schemas), server entry point, vasig mcp command, integration test. Fixed Zod schema registration bug caught in code review.
