---
id: v-udgs
status: closed
deps: []
links: []
created: 2026-03-22T10:24:50Z
type: chore
priority: 1
assignee: Jibles
tags:
    - cli
    - signing
    - agent
---
# CLI: Suppress SDK console logging during signing

## Objective

The SDK logs 11 emoji-prefixed progress lines to stdout during every `vault.sign()` call. These pollute structured output (`--output json` becomes unparseable) and waste agent context window tokens. Suppress them entirely and optionally emit our own minimal progress.

## Context & Findings

### The problem

During MPC signing, the SDK emits hardcoded `console.log` calls:
```
🔑 Generated signing party ID: sdk-6935
📡 Calling FastVault API with session ID: b351d96e-...
✅ Server acknowledged session: ...
⏳ Waiting for server to join session...
✅ All participants ready: [sdk-6935, Server-253]
📡 Starting MPC session with devices list...
✅ MPC session started
🔐 Starting MPC keysign process...
🔏 Signing message: c18fd349...
✅ Signature obtained for message
🔄 Formatting signature results...
```

In `--output json` mode, these lines appear on stdout before the JSON blob, making `JSON.parse(stdout)` fail. This was confirmed as a P0 bug in send testing.

### Analysis of the log content

None of these lines contain information useful to the CLI consumer:
- Session IDs, party IDs, message hashes are internal SDK state
- "Waiting for server" is the only human-useful line (indicates progress)
- "Signature obtained" is redundant — the result tells you this
- The SDK already has a proper `signingProgress` event system with structured `SigningStep` objects

### Solution

Monkey-patch `console.log` to `/dev/null` during signing operations. The SDK fix (log level/suppression option) is tracked in `vc-mpnw` but we need this workaround now.

Optionally emit our own single progress line to stderr: `Signing transaction...` (so humans know something is happening during the 5-60s wait).

## Files

- `src/lib/signing.ts` — wrap the `signWithRetry` function to suppress `console.log` during the signing call and restore after
- `src/commands/send.ts` — optionally emit `Signing transaction...` to stderr before the sign call
- `src/commands/swap.ts` — same

## Acceptance Criteria

- [ ] No SDK emoji log lines appear on stdout during signing
- [ ] `vasig send --output json` returns valid parseable JSON on stdout (no interleaved log lines)
- [ ] `vasig swap --output json` returns valid parseable JSON on stdout
- [ ] A single progress line like `Signing transaction...` appears on stderr (optional, for human feedback)
- [ ] `console.log` is restored after signing completes (success or failure)
- [ ] Other non-signing `console.log` calls in the codebase (if any) are unaffected
- [ ] Swap command emits quote + result as a single JSON object in `--output json` mode (not two separate blobs)
- [ ] Lint and type-check pass

## Gotchas

- Must restore `console.log` in a `finally` block — if signing throws, the original must be restored
- The `signWithRetry` wrapper in `src/lib/signing.ts` is the ideal place since all signing goes through it
- `console.warn` and `console.error` should NOT be suppressed — only `console.log`
- Test that the retry path also suppresses correctly (the retry message `⚠ MPC signing timed out, retrying...` is from our code via `process.stderr.write`, so it's fine)
