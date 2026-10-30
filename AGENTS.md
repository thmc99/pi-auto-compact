# AGENTS.md

Pi extension: before sending a prompt, check context usage as a percentage of the model's window. If it is over the threshold, run `ctx.compact()` first, then send. During a run, `tool_call` only reports context pressure; mid-run compaction is left to Pi.

This is a personal fork of `Chasen-Liao/pi-auto-compact` (remote `upstream`). It is not published to npm and is not intended to be merged back. The only functional change from upstream is the minimum threshold: 20 instead of 30.

The only source file is `extensions/auto-compact.ts`. There is no build step. Peer dependency: `@earendil-works/pi-coding-agent >=0.99.0` (Node `>=22.19.0`). Types are checked against 1.0.2; the end-to-end baseline is still 0.99.1.

## Commands

```bash
npm run typecheck
npm test                 # test/smoke.ts; PI_CODING_AGENT_DIR points at a temp dir, cleaned up afterwards
pi -e .                  # load the extension locally
```

After changing `peerDependencies`, `devDependencies` or `engines`, run `npm install --package-lock-only`. Otherwise `npm ci` goes out of sync with the lockfile.

## Installing the fork

Install from git or a local path instead of the npm package, and remove `npm:pi-auto-compact` from `packages` in `~/.pi/agent/settings.json` so the two don't both load. Run `/reload` after any change.

## Conventions

- Compaction is only allowed in the idle `input` preflight, via `compactAndWait`. **Never** call `ctx.compact()` from callbacks that run during a turn (`tool_call`, `turn_end`, `agent_end`, etc.). It calls `abort()` first, which discards the whole turn and loses the prompt.
- Mid-run compaction belongs to Pi: between tool batches, `shouldCompact` fires, Pi compacts, then continues. Do not reimplement it. The formula is `tokens > window - reserveTokens`, where `reserveTokens` resolves as per-model override, then global setting, then 16384.
- Aligning thresholds: write the per-model `reserveTokens` directly into `~/.pi/agent/settings.json` (the API is read-only). Pi only picks it up after `/reload`. Only touch keys this extension wrote itself; skip any reserve the user set. Side effect: a larger reserve also raises the summary token cap, `min(0.8 × reserve, model.maxTokens)`, so summaries may get longer.
- `tool_call` only reads usage and reports status. It never blocks and never compacts.
- `turn_end` must return `undefined`. Returning `{continue: true}` causes a spurious extra turn.
- Do not re-send the original prompt with `sendUserMessage`. `input` runs inside `prompt()`, so that would recurse. Return `{action: "continue"}` instead.
- Async callbacks: guard with `sessionGeneration` and wrap UI calls in try/catch. Soft errors ("Nothing to compact", "Already compacted") let the prompt through. Hard errors block it, and the user recalls it with the up arrow and resubmits.
- No compaction timeout fallback. Decided in 1.2.2: every current `before_compact` handler returns synchronously, so that path is unreachable.
- Config file `~/.pi/agent/pi-auto-compact.json` (resolved via `getAgentDir()` / `PI_CODING_AGENT_DIR`):
  - `threshold` must be in `[20, 99)`.
  - `alignPiThreshold` defaults to false.
  - `alignedReserveTokens` records the reserves this extension wrote.
  - An out-of-range value falls back to the last valid one.
  - Reads and writes hot-reload.
  - Writes go through `updateJsonFile`, an atomic merge write.

## Verification and limits

After any change, run `npm run typecheck` and `npm test` (24 mock smoke tests). For end-to-end checks, use `pi -p` with an isolated cwd and an isolated `PI_CODING_AGENT_DIR`; `pi -c` would attach to the newest session in the same directory. After upgrading Pi, re-test `ctx.compact()`, `input`, `tool_call` and `shouldCompact`.

Not preflighted:
- Models with no `contextWindow`.
- steer / followUp messages.
- Growth from skill or template expansion (Pi covers this).
- `tool_call` does not predict tool results that haven't been produced yet.

Recipe to reproduce "compact between batches, then continue" (isolated dirs): set `reserveTokens=270000` and `keepRecentTokens=1000`, then run one very large bash output.

## Status

Based on upstream 1.3.1. Types verified against pi 1.0.2 and the compaction contract is unchanged. The isolated end-to-end run has not been repeated.
