# sign-safe — offline signing-time safety gate for Solana

[![npm version](https://img.shields.io/npm/v/sign-safe.svg)](https://www.npmjs.com/package/sign-safe)
[![CI](https://github.com/lrafasouza/sign-safe-skill/actions/workflows/ci.yml/badge.svg)](https://github.com/lrafasouza/sign-safe-skill/actions/workflows/ci.yml)
[![license: MIT](https://img.shields.io/npm/l/sign-safe.svg)](https://github.com/lrafasouza/sign-safe-skill/blob/main/LICENSE)

> Submission for the **Solana AI Kit** skill bounty. Not a link-only placeholder: a runnable, tested, **npm-published** skill. The full source lives in the [`sign-safe-skill`](./sign-safe-skill) submodule (`lrafasouza/sign-safe-skill`); install it directly as [`sign-safe`](https://www.npmjs.com/package/sign-safe). Kit integration: [solana-ai-kit#34](https://github.com/solanabr/solana-ai-kit/pull/34).

**What does this Solana transaction do to *me* — before I sign it?** Wallets show a blob. Auditors read code you already wrote. Debuggers explain failures after they land. None answer the question that matters at signing time. sign-safe does.

Hand it opaque base64 transaction bytes. It decodes them **offline** — legacy + v0 + Address Lookup Tables, durable nonces, Token-2022, Squads v4 proposals — classifies danger primitives, computes signer-perspective outflow, and emits a verdict with the CLI exit code mirroring it:

| Verdict | Exit | Meaning |
|---|---|---|
| **SIGN** | `0` | Recognized benign shapes under policy. *Not* a guarantee of intent. |
| **HOLD** | `10` | Needs a human / higher-trust policy before signing. |
| **REJECT** | `20` | Known dangerous shape, or bytes that can't be trusted. |

## Try it (no clone)

```bash
npx sign-safe <tx.b64>          # 0 SIGN · 10 HOLD · 20 REJECT
cat tx.b64 | npx sign-safe      # or pipe base64 on stdin
npx sign-safe --json <tx.b64>   # verdict.json only, for agents

npx sign-safe-mcp               # zero-dependency MCP server (tool: review_transaction)
```

For Claude / agents: the skill exposes a `/sign-review` command and an MCP `review_transaction` tool with a published `outputSchema`, plus a `guardedSignTransaction` wrapper that throws on REJECT *before* the key is touched. No key, no RPC, no broadcast — the deterministic core is offline by design.

## Real output

```text
$ npx sign-safe safe-transfer.b64
[ SIGN ]  Recognized instructions within thresholds; no danger primitives, unknown
          programs, or unverified ALT references. sends 10000000 lamports to 9hSR6S7W…
          Not a guarantee of intent — verify the recipients and amounts yourself.
worst severity: INFO · static outflow: 10000000 lamports · findings: 0        (exit 0)

$ npx sign-safe setauthority.b64
[ REJECT ]  Contains a REJECT-class danger primitive: SPL Token SetAuthority.
findings (1):
  - [REJECT] ix#0 SPL Token SetAuthority
      maps to loss: Hands mint/freeze/owner authority to an attacker, who can then
      mint, freeze, or seize at will.                                          (exit 20)
```

## What it catches (the shapes simulation/balance-diff checks miss)

A danger move at signing time often moves **zero** tokens — so wallet simulation shows nothing. sign-safe reads the *intent* in the bytes:

- **Owner reassignment** — System `Assign`
- **Token authority handoff** — SPL `SetAuthority` (mint / freeze / account owner)
- **Unlimited delegate** — SPL `Approve`
- **Account close / rent drain**, and NFT-theft shapes
- **Durable-nonce replay** — the documented 2026 Drift (~$285M) vector
- **Program upgrade-authority swap** — BPF loader TOCTOU
- **Token-2022 traps** — permanent-delegate, transfer-hook
- **Squads v4 proposals** — decodes the *hidden inner instruction* and names it
- **ALT-obscured accounts / unknown programs / unresolved references** → fail-closed (never silent SIGN)

A 14-program clear-signing registry (Jupiter, Orca, Raydium, Kamino, Drift, …) recognizes benign DeFi/NFT instructions so they don't drown the real signals; everything unrecognized stays conservative.

## Why it's different

- **Real on-chain attack evidence.** The actual transactions that drained Drift (~$285M, Apr 2026) decode to **HOLD** *before* signing — durable-nonce advance + Squads v4 execution. That's the exact authority-handoff shape balance-diff checks miss, because nothing needs to move at signing time for the transaction to be dangerous.
- **Honest by construction.** SIGN is the *weakest* claim in the system ("nothing here is recognized as dangerous"), never "this is safe." A banned-phrase contract is executed over every verdict. Malformed / unknown / unresolved structure fails closed into HOLD/REJECT.
- **Tested, reported as-is.** 803 tests / 42 files · registry discriminators **69/69** cross-verified against canonical on-chain IDLs · mutation testing (Stryker, 70.2% behavioral) + fuzz (44,288 single-byte mutations of an authority transfer → **0** ever flip to SIGN) · 500 benign mainnet txs → 18.4% SIGN / 81.6% HOLD / **0 false-REJECT** · dual-parser cross-validation (@solana/web3.js v1 + @solana/kit v2) · **zero runtime dependencies** · MIT.

## Scope (honest limits)

Pre-sign review, **not** on-chain enforcement. The 37/37 attack-replay and 100%-recall figures are over a curated, mostly-synthetic corpus plus 2 real Drift transactions — not a population sample. sign-safe complements simulation, wallet UX, human review, and multisig/policy systems; it is the byte-level static layer *before* the signing function is allowed to run.

## Evaluate it

```bash
# one line, no clone:
npx sign-safe <tx.b64>

# or the full suite from the public repo:
git clone https://github.com/lrafasouza/sign-safe-skill
cd sign-safe-skill && npm ci && npm run verify:all && npm run demo:attack-pack
# → 803 passed · 80 fixtures PASS/0 FAIL · 37/37 attacks held · False SIGN: 0
```

## Links

- 📦 npm: https://www.npmjs.com/package/sign-safe
- 📂 Repo (full docs, danger catalog, precision report, 3-minute evaluator quickstart): https://github.com/lrafasouza/sign-safe-skill
- 🔗 Kit integration PR: https://github.com/solanabr/solana-ai-kit/pull/34

Full documentation lives in the [`sign-safe-skill`](./sign-safe-skill) submodule's `README.md`, `SKILL.md`, `SECURITY.md`, and `docs/`.
