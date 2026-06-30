# sign-safe — offline signing-time safety gate for Solana

> Submission for the **Solana AI Kit** skill bounty. The skill lives in the [`sign-safe-skill`](./sign-safe-skill) submodule (`lrafasouza/sign-safe-skill`) and is published to npm as [`sign-safe`](https://www.npmjs.com/package/sign-safe). Kit integration: [solana-ai-kit#34](https://github.com/solanabr/solana-ai-kit/pull/34).

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
npx sign-safe-mcp               # zero-dependency MCP server for agents
```

No key, no RPC, no broadcast — the deterministic core is offline by design.

## Why it's different

- **Real on-chain attack evidence.** The actual transactions that drained Drift (~$285M, Apr 2026) decode to **HOLD** *before* signing — durable-nonce advance + Squads v4 execution. That's the exact authority-handoff shape simulation/balance-diff checks miss, because nothing needs to move at signing time for the transaction to be dangerous.
- **Honest by construction.** SIGN is the *weakest* claim in the system ("nothing here is recognized as dangerous"), never "this is safe." Malformed, unknown, or unresolved structure fails closed into HOLD/REJECT — never silent approval.
- **Tested, reported as-is.** 803 tests / 42 files · 14-program clear-signing registry with **69/69** Anchor discriminators cross-verified against canonical on-chain IDLs · mutation testing (Stryker, 70.2% behavioral) + fuzz (44,288 single-byte mutations of an authority transfer → **0** ever flip to SIGN) · 500 benign mainnet txs → 18.4% SIGN / 81.6% HOLD / **0 false-REJECT**. Zero runtime dependencies. MIT.

## Scope (honest limits)

Pre-sign review, **not** on-chain enforcement. The 37/37 attack-replay and 100%-recall figures are over a curated, mostly-synthetic corpus — not a population sample. sign-safe complements simulation, wallet UX, human review, and multisig/policy systems.

## Links

- 📦 npm: https://www.npmjs.com/package/sign-safe
- 📂 Repo (full docs, danger catalog, 3-minute evaluator quickstart): https://github.com/lrafasouza/sign-safe-skill
- 🔗 Kit integration PR: https://github.com/solanabr/solana-ai-kit/pull/34

Full documentation is in the [`sign-safe-skill`](./sign-safe-skill) submodule's `README.md` and `SKILL.md`.
