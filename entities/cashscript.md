
---
pageType: entity
entityType: toolchain
id: entity.cashscript
description: CashScript - high-level language for BCH smart contracts. Compiles to Bitauth Auth Templates or libauth-compatible bytecode. The official SDK for writing covenant contracts.
sourceUrl: https://cashscript.org
---

# CashScript

> CashScript — high-level language for BCH smart contracts. Compiles to Bitauth Auth Templates or libauth-compatible bytecode. The official SDK for writing covenant contracts.

**Project:** [cashscript.org](https://cashscript.org)
**Compiler:** `cashc` **v0.13.0** (pinned by ParyonUSD for artifact stability)
**Library:** `@cashscript/cashscript` + the third-party stack `@cashlab/common` + `@cashlab/cauldron` by hosseinzoda

## What CashScript is

A Solidity-like language for Bitcoin Cash. The compiler emits either:

1. A **Bitauth Authentication Template** (JSON, editable in [ide.bitauth.com](https://ide.bitauth.com)) — for human authorship and IDE-driven development.
2. **libauth-compatible bytecode** — for direct integration with TypeScript wallets.

The runtime library (`cashscript`) is the equivalent of `web3.js` for BCH covenants: it imports compiled artifacts, instantiates `Contract` objects, and calls covenant functions. It is the toolchain the bot uses if it ever needs to *interact* with a covenant at a known address (rather than just sending plain BCH).

## The compiler: `cashc 0.13.0`

ParyonUSD pins `^0.13.0` in every `.cash` file's `pragma cashscript ^0.13.0;`. This is intentional: bytecode is **stable across 0.13.x patch versions for a fixed `.cash` source** (verified in `paryonusd/contracts/contracts/PriceContract.cash` and `Borrowing.cash`). What changes between patch versions is **debug metadata** (source mappings). Always pin `cashc` to a known version and verify the artifact **fingerprint** to confirm bytecode integrity.

**Artifact shape** (TypeScript module, JSON-equivalent):

```ts
export default {
  contractName: 'PriceContract',
  constructorInputs: [
    { name: 'migrationKey', type: 'bytes' },
    { name: 'oraclePubKey', type: 'pubkey' },
    { name: 'initialPrice', type: 'int' },
  ],
  abi: [
    { name: 'updatePrice', inputs: [
      { name: 'priceInfo', type: 'datasig' },
      { name: 'newPrice', type: 'int' },
    ]},
    { name: 'migrateContract', inputs: [
      { name: 'newBytecode', type: 'bytes' },
      { name: 'migrateSig', type: 'sig' },
    ]},
  ],
  bytecode: 'OP_3 OP_PICK OP_3 OP_ROLL OP_3 OP_PICK OP_HASH256 ...',
  source: '... full source ...',
  fingerprint: 'ce9e4176f4bf4d5...sha256 of bytecode',
  compiler: { name: 'cashc', version: '0.13.0' },
};
```

The `fingerprint` is the SHA-256 of the **bytecode** (not the source). If two artifacts have identical bytecode and identical fingerprints, they are the *same contract*, period. ParyonUSD's bytecode-verification.md walks through how to verify the published artifact's fingerprint against an audit snapshot.

## The library: `@cashscript/cashscript`

- Imports the artifact (`.ts` or `.json`).
- Instantiates with constructor parameters: `new Contract(artifact, [migrationKey, oraclePubKey, initialPrice], { provider: networkProvider })`.
- Function calls are constructed lazily: `contract.functions.updatePrice(priceInfo, newPrice).send(...).to(...)`.
- `.to(provider)` returns a TransactionBuilder; `.from(...)` adds inputs (UTXOs in the contract's locking bytecode); `.withOpReturn(...)`, `.withTime(...)`, `.withFeePerByte(...)`.
- `.send()` returns a TransactionResult with `.txid`.

The library handles UTXO selection, change output construction, and CashTokens-aware inputs automatically *only* for inputs in the contract's locking script. The bot usually still needs to compose the spend by hand because the bot's UTXOs come from P2PKH addresses, not covenant addresses.

## The third-party stack: `@cashlab/common` + `@cashlab/cauldron`

`@cashlab/cauldron` (hosseinzoda) wraps the **Cauldron Liquidity Pool** contract as an idiomatic TypeScript SDK. The pool itself is **not** published as a `.cash` file — it's published as a **Bitauth Auth Template** (`cauldron-libauth-template.json`) and the runtime bytecode is generated programmatically.

Why: the pool's locking script is parameterized by the **owner's `withdraw_pubkey_hash`** (a 20-byte RIPEMD-160/SHA-256 hash). Without this hash you can't compile the contract. The Cauldron architecture therefore has the SDK produce the bytecode on the fly:

```ts
const poolContract = await ExchangeLab.generatePoolV0LockingBytecode(
  withdrawPubkeyHash // Buffer, 20 bytes
);
```

For ParyonUSD the situation is different: the contracts are published as .cash, the artifacts are published as TS modules (`@paryonusd/contracts`), and **fingerprint-based verification is the integrity check** — you check `artifact.fingerprint` against the published audit hash before doing anything.

So the toolchain layout is:

| Contract family | Source | Artifact | Verification          |
|-----------------|--------|----------|-----------------------|
| PUSD / EarnVault-style | `.cash` files | `paryonusd/contracts/artifacts/*.ts` modules | `fingerprint` |
| Cauldron pool           | Auth Template JSON | `@cashlab/cauldron` SDK | `ExchangeLab.generatePoolV0LockingBytecode(hash)` is the canonical bytecode source |
| Custom contracts (you) | `.cash` you write | compiled with `cashc` | your own fingerprint registry |

## Pitfalls

1. **Bytecode is stable across 0.13.x patch versions for a fixed .cash source, but debug metadata changes.** Always pin `cashc` to a specific version. If you bump `cashc`, expect every artifact to recompile; `fingerprint` will be the same iff bytecode is the same, but `source` mappings and ABI might shift.
2. **The `^0.13.0` pragma means "compatible within 0.13.x" — minor version bumps to 0.14.0 will fail to compile.** If you want to bump deliberately, change the pragma AND bump the compiler, AND verify fingerprints.
3. **Cauldron pools are NOT .cash files.** The published "source" is a Bitauth Auth Template in `cashlab/packages/cauldron/src/cauldron-libauth-template.json`. The SDK generates bytecode at instantiation time. There's no canonical `.cash` source.
4. **`cashc` 1.x is the major-version-bump series** as of 2024–2026; if you see 1.x artifacts in the wild, treat them as not cross-compatible with 0.13.x. The PUSD artifacts will not run under cashc 1.x without re-compilation.
5. **The `migrateContract()` function exists for upgradeable contracts.** If you ship a contract with `migrateContract`, the operator's `migrationKey` can replace the bytecode at any time. ParyonUSD ships this on `PriceContract` — it's the one mutable contract in the system. Don't accept a "ParyonUSD-compiled" PriceContract artifact from a third party without checking the published `migrationKey` (which lives in `paryon-verify/config.ts`).
6. **`Contract.fromCashScript()` does not exist.** The library is `new Contract(artifact, args, opts)`. Misnaming it during refactors is common.
7. **`@cashlab/cauldron`'s contract is pool-v0 specific.** A v1 of the pool would have a different auth template — make sure your code reads `cauldron_poolv0` (the template name) explicitly.
8. **CashScript covenants can deadlock liquidity.** A covenant that requires a *signature* of the contract holder means the holder's pubkey must always be available — losing the pubkey loses the funds. ParyonUSD's Borrowing contract is structured so the holder is a sidecar contract, not a real person; Cauldron's pool is structured so the withdrawer is the owner. Always check who can publish a key before depositing funds.
9. **CashScript fees default to 1.0 sat/byte** in cashscript.ts's `Contract.withFeePerByte`. Override if the network is congested, but be aware: BCH fee markets rarely go above 5 sat/byte — most bots never override this.
10. **No formal verifier.** CashScript has test fixtures and coverage guides (ParyonUSD ships `post-audit-changes.md` and `bytecode-verification.md`) but no Foundry-style fuzzing like Ethereum Vyper does. External audit + fingerprint verification is the de-facto standard.

## Companion libraries

- `@cashlab/common` — shared helpers used by `@cashlab/cauldron`. Pure-TS, types-first.
- `@cashlab/moria` — for mortar/couriers (cross-contract patterns).
- `hosseinzoda/vegabch` — a CLI for interacting with CashScript contracts (cross-platform Node/npm). Useful for ops.
- `hosseinzoda/bchcockpit` — a browser-only withdrawal UI hosted at `hosseinzoda.github.io/bchcockpit/`. Read-only or limited-spend; convenient as a manual fallback.

## Related entities

- `entity.libauth` — provides the runtime types and signing path cashc's emitted bytecode relies on.
- `entity.paryonusd` — the PUSD ecosystem compiled with cashc 0.13.0.
- `entity.cauldron-dex` — uses `@cashlab/cauldron` for the pool template, *not* a .cash file.
- `concept.cash-tokens` — the FT/NFT primitives cashc generates bytecode against.
