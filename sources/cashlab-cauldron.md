
---
pageType: source
sourceType: code
sourceUrl: https://github.com/hosseinzoda/cashlab
title: cashlab - Bitauth Auth Template + TS SDK for Cauldron DEX
accessedAt: 2026-09-17
description: The canonical source for the cauldron_poolv0 Bitauth Authentication Template and the @cashlab/cauldron TypeScript SDK. Backed up by inspecting the Auth Template JSON and the TypeScript SDK util.ts.
---

# cashlab — Cauldron DEX Bitauth Template + SDK

> Canonical source for the **cauldron_poolv0** Bitauth Authentication Template (`packages/cauldron/src/cauldron-libauth-template.json`) and the `@cashlab/cauldron` TypeScript SDK (`packages/cauldron/src/util.ts`).

**Repository:** [github.com/hosseinzoda/cashlab](https://github.com/hosseinzoda/cashlab)
**License:** Apache-style (per the repository)
**Accessed:** 2026-09-17 by subagent deleg_e72313b5/task-5 via `git clone --depth=1`
**Author:** hosseinzoda (Hossein Zoda)

## What cashlab is

The `@cashlab/cauldron` package is the TypeScript-side surface for the **Cauldron DEX Liquidity Pool** contract. Riften Labs' `docs.riftenlabs.com/cauldron/` is the user-facing documentation; cashlab is the *runtime* — what an integrator imports to construct pool bytecode, build a swap transaction, or query trade fee math. It also ships the Auth Template that ships under `cauldron_poolv0`.

## Path map

```
cashlab/
  packages/
    cauldron/
      src/
        cauldron-libauth-template.json     <-- the Auth Template for the pool
        util.ts                            <-- fee math (calcTradeFee) + helpers
        exchange-lab.ts                    <-- generatePoolV0LockingBytecode, constructTrade*, verifyTrade*
        tests/                             <-- TS fixtures and trade scenarios
        README.md
      package.json
      tsconfig.json
      ...
    common/
      src/
        ...                                <-- shared helpers used by cauldron
    moria/
      ...                                 <-- mortar / courier patterns
```

## The pool bytecode

The Cauldron V0 pool is **not** published as `.cash`. It is a Bitauth Authentication Template (`cauldron_libauth_template.json`) with two scripts in the `pool_owner` entity:

- `cauldron_poolv0` — the **main pool** script. Anyone can spend IF the input satisfies the k-invariant (`out_sats_a / in_sats_a == out_sats_b / in_sats_b` modulo the fee constant). This is the swap/exchange branch.
- `withdraw_cauldron_poolv0` — the **owner-only withdrawal branch**. Owner signs with `pool_owner_key.schnorr_signature.all_outputs` against `pool_owner_key.public_key`. OP_DEPTH branches the two: stack-depth 1 means "just withdrawal", stack-depth 2 means "trade".

**Pattern:** `OP_DEPTH OP_IF <withdraw branch> OP_ELSE <swap branch> OP_ENDIF`. Each branch is the libauth template's "unlocks" property pointing at the corresponding script.

**Pool instantiation:** the only public function for generating bytecode is

```ts
ExchangeLab.generatePoolV0LockingBytecode(withdraw_pubkey_hash: Uint8Array)
```

where `withdraw_pubkey_hash` is a **20-byte hash** (RIPEMD-160 of the owner's compressed public key). Pass the wrong length and you get invalid script.

## Trade fee math

The on-chain fee is **`amount * 3 / 1000`** = **0.3%** in BCH (or whichever asset is the trade side paying the fee). Verified in `packages/cauldron/src/util.ts`:

```ts
export const calcTradeFee = (a: bigint) => (a * 3n) / 1000n;
```

This is the script-encoded literal `OP_MUL <3> OP_MUL <1000> OP_DIV` — denominator is 1000, not 100, so the fee is **0.3%, not 3%**. Riften's docs (live page) say *"fee = abs(satoshis of input - satoshis of outputs) \* 0.03"* — that `0.03` is a typo for `0.003`. Always trust the bytecode (`*3 /1000`).

## Trading API

`ExchangeLab` (called via the `@cashlab/cauldron` SDK) exposes:

- `constructTradeBestRateForTargetDemand(targetSats, pool, feePayoutToken)` — finds the pool that gives the best rate for a given demand size; routes across multiple pools if needed.
- `constructTradeAvailableAmountBelowTargetRate(...)` — given a target rate, what's the maximum amount available?
- `createTradeTx(args)` — builds the unsigned transaction consuming pool UTXOs and producing new pool UTXOs.
- `verifyTradeTx(args)` — re-checks the math (k-invariant + fee) on the resulting tx; the SDK runs this before broadcasting as a sanity check.

These functions operate **on the token UTXO** — they accept `tokenA` + `tokenB` amounts as bigints and return the trade result. They do *not* sign the transaction; signing is the consumer's job.

## Companion projects

| Project            | URL                                                       | Purpose                                                            |
|--------------------|-----------------------------------------------------------|--------------------------------------------------------------------|
| `hosseinzoda/vegabch` | https://github.com/hosseinzoda/vegabch                  | CLI for interacting with CashScript contracts; `npm install -g vegabch` |
| `hosseinzoda/bchcockpit` | https://github.com/hosseinzoda/bchcockpit            | Browser-only withdrawal UI hosted at `hosseinzoda.github.io/bchcockpit/` |
| `hosseinzoda/cashlab` (this repo) | https://github.com/hosseinzoda/cashlab          | SDK + Auth Template source                                         |
| Riften Labs docs | https://docs.riftenlabs.com/cauldron/                     | User-facing documentation; not authoritative on bytecode internals |

## Pitfalls

1. **The fee notation in `docs.riftenlabs.com/cauldron/swap/` is misleading.** It says `* 0.03`, suggesting 3%. The on-chain reality is `* 3 / 1000 = 0.3%`. Use the SDK's `calcTradeFee` or the literal VM opcodes as the source of truth.
2. **`generatePoolV0LockingBytecode`** takes a 20-byte hash, not a pubkey. Pass a 33-byte pubkey and you get broken script. RIPEMD160 first.
3. **The pool template's name is `cauldron_poolv0`**, not `cauldron_pool` — if a future version introduces `cauldron_poolv1`, be sure to pin to `v0` explicitly.
4. **There is no Cauldron deposit function** in V0. Pool liquidity is created by the pool-owner in a genesis transaction. Anyone can trade, only the owner can withdraw.
5. **`@cashlab/cauldron`'s number type is `bigint`** throughout — passing `Number` where `bigint` is expected silently breaks on trade sizes > 2^53.

## Related entities

- `entity.cashscript` — for the parent toolchain.
- `entity.cauldron-dex` — for the user-facing DEX concept.
- `entity.libauth` — provides the runtime types the cashlab SDK builds against.
- `entity.paryonusd` — different covenant family, but shares the cashc 0.13.0 + CashScript-friendly architecture.
