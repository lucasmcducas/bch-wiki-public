---
pageType: entity
entityType: software
id: entity.selene-wallet
description: Selene Wallet - BSD-3 licensed BCH wallet (last commit 2026-09-29) shipped as a React/Capacitor mobile+web app (iOS/Android via Capacitor, browser web build). Canonical reference for CashTokens-aware wallet kernel: KeyManager, AddressManager, ElectrumService, TransactionBuilder, UtxoManager, TokenManager. Pattern source for the bch-bot build.
sourceUrl: https://gitlab.com/selene.cash/selene-wallet
mirror: https://git.xulu.tech/selene.cash/selene-wallet
---

# Selene Wallet

> Selene Wallet — BSD-3 licensed, actively maintained (last commit **2026-09-29**) BCH wallet by Kallisti. Ships with full CashTokens (FT + NFT) support, ships on mobile (iOS/Android via Capacitor) and as a browser web build. **Canonical reference for any headless BCH wallet bot**: the kernel services in `src/kernel/` are well-architected and can be re-implemented as Node.js scripts without the React/Capacitor/Redux UI layer.

> **Correction (2026-10-01): there is no desktop/Electron build.** This page previously said "ships on desktop (Electron) and mobile". Verified against `main` HEAD `48237e6`: no Electron dependency in `package.json`, no `src-electron/` directory, and the only occurrences of the string "electron" in the whole tree are two commented-out Electrum server hostnames in `src/util/network.ts`. `README.md` gives build instructions for Android and iOS only, and [selene.cash](https://selene.cash/) offers "Try Web Version" and "Download .apk" — no desktop installer. This matters for security, not just accuracy: the web build's encryption is a **passthrough** (see [`../security/wallet-key-storage.md`](../security/wallet-key-storage.md) §3.1), so there is no hardened-desktop tier to point desktop users at.

**Repository:** [gitlab.com/selene.cash/selene-wallet](https://gitlab.com/selene.cash/selene-wallet) (mirrored at [git.xulu.tech/selene.cash/selene-wallet](https://git.xulu.tech/selene.cash/selene-wallet))
**License:** BSD-3-Clause
**Last verified:** 2026-09-17
**Default branch:** `main` (NOT `master`)
**Sources:** Live `main` branch, `package.json`, `src/util/network.ts`, `src/kernel/bch/*.ts`, `src/kernel/wallet/*.ts`

## What Selene is (and isn't)

A real, deployed BCH wallet with CashTokens support. Has a React UI, Capacitor for mobile, Redux for state, Apollo/GraphQL client, Ant Design components. Do NOT fork the whole app for a bot — the UI is ~70% of the code and tightly coupled.

What you want from Selene: the **kernel services** (`src/kernel/`) which are the actual wallet logic, in TypeScript. They total ~4,500 LOC and demonstrate every BCH + CashTokens operation correctly. The bch-bot build re-implements moth-style CLI scripts that mirror Selene's kernel design.

## Server list (verified 2026-09-17, all reachable)

Selene's `src/util/network.ts` ships canonical server endpoints that the bch-bot uses directly:

**Mainnet:**
- `cashnode.bch.ninja:50004` — Kallisti / Selene Official
- `bitcoincash.network:50004` — Dagur
- `blackie.c3-soft.com:50004` — Calin
- `bch.loping.net:50004`
- `bch.soul-dev.com:50004`
- `bitcoincash.stackwallet.com:50004` — Stack Wallet
- `node.minisatoshi.cash:50004` — minisatoshi
- `fulcrum.criptolayer.net:50004` — molecular

**Chipnet:**
- `chipnet.bch.ninja:50004` — Kallisti
- `chipnet.c3-soft.com:64004` — Calin

**Cauldron (CashTokens-aware Rostrum):**
- `rostrum.cauldron.quest:50004`

**Testnet3:**
- `blackie.c3-soft.com:60004`

**Testnet4:**
- `blackie.c3-soft.com:62004`
- `tbch4.loping.net:62004`

`bch.imaginary.cash:50004` (mainnet) and `chipnet.imaginary.cash:50004` (chipnet) are documented in Selene but **dead as of 2026-09-17** — TCP connection refused. Do not use.

## Protocol versions

Selene declares two protocol version constants:
- `ELECTRUM_PROTOCOL_VERSION = "1.5"`
- `ROSTRUM_PROTOCOL_VERSION = "1.4.3"`

**CORRECTION (2026-10-01) — the claim below was wrong and it shipped as a bug.**

An earlier version of this page read: *"all reachable servers actually report protocol `1.5`; use `1.5` for both; negotiating `1.4.3` fails with `unsupported protocol version` on every server tested."* That is the inverse of what the mainnet Fulcrum 2.1.2 nodes do, and following it is what produced the bug recorded in `syntheses/failure-modes-we-hit.md` §15.

Measured directly on `cashnode.bch.ninja:50004` over TLS+WebSocket:

| Requested | Result |
|---|---|
| `["1.4","1.4.3","1.5"]` | `ERROR Unsupported protocol version` |
| `["1.4","1.4.3"]` | `["Fulcrum 2.1.2","1.4.3"]`, real data returned |
| `["1.5"]` | `ERROR Unsupported protocol version` |

Asking for `1.5` does not fail loudly. The socket opens, `server.version` answers plausibly, `blockchain.scripthash.*` keeps working, and every other `blockchain.*` call returns an **empty object `{}`** — so balance reads zero and swaps fail with `Missing inputs`, with no error anywhere.

**Use `electrum: '1.4', rostrum: ['1.4','1.4.3']`** — offer the list, newest first, so a genuinely newer node can still select a newer version. The original "verified" note was most likely taken from Selene's *constants* (`"1.5"`) or from a server's reported `server.version` string, rather than from a successful transaction fetch; a version string is not proof a version works.

## Kernel services (the parts worth borrowing)

Located in `src/kernel/` (Selene source tree):

| File | LOC | What it does | bch-bot equivalent |
|------|-----|--------------|---------------------|
| `wallet/WalletManagerService.ts` | 589 | Multi-wallet CRUD, BIP39 passphrase, encryption (Capacitor SimpleEncryption), SqlJsDatabase-backed metadata | none — bch-bot is single-wallet, no DB |
| `wallet/KeyManagerService.ts` | 264 | BIP39→HD node→address derivation; libauth compiler-based signing; `signInputs()` for P2PKH; manual `signTemplate()` for CashScript contracts | `lib/wallet.mjs` (BIP39+HD only); `lib/sign.mjs` (P2PKH signing) |
| `wallet/AddressManagerService.ts` | 410 | Address book, generation, gap-limit scan, address labels | `lib/wallet.mjs::deriveReceivingAddresses` (subset) |
| `wallet/AddressScannerService.ts` | 503 | Wallet discovery via Electrum subscription; transaction history backfill | not ported |
| `wallet/UtxoManagerService.ts` | 353 | UTXO tracking with CashTokens awareness | `scripts/utxos.mjs` (read-only) |
| `wallet/TokenManagerService.ts` | 255 | FT/NFT aggregation, minting-baton tracking | deferred to Phase 2 |
| `bch/ElectrumService.ts` | 673 | Electrum client wrapper, version negotiation, server failover, blacklist | `lib/network.mjs` |
| `bch/TransactionBuilderService.ts` | 909 | Build P2PKH tx, stablecoin tx, token tx with fee-paying token category; BIP69+CashTokens output sort | `lib/sign.mjs` (P2PKH only); Phase 2 adds token tx |
| `bch/TransactionManagerService.ts` | 521 | Broadcast, history, mempool tracking | not ported (moth uses `blockchain.transaction.broadcast` directly) |

**Total kernel:** ~4,477 LOC. **bch-bot equivalent:** ~681 LOC, single-purpose scripts. The bot omits multi-wallet, encryption, history backfill, and any UI integration.

## The signing pattern (P2PKH, no CashTokens)

Selene's `KeyManagerService.signInputs` is the model the bch-bot's `lib/sign.mjs` follows. The whole flow is ~30 lines because **libauth's compiler handles the signing serialization**:

```ts
// Pseudocode of Selene's approach
const template = importWalletTemplate(walletTemplateP2pkhNonHd);
const compiler = walletTemplateToCompilerBCH(template);

const signedInputs = inputs.map((input) => ({
  outpointTransactionHash: hexToBin(input.tx_hash),
  outpointIndex: input.tx_pos,
  sequenceNumber: 0,
  unlockingBytecode: {
    compiler,
    script: "unlock",
    valueSatoshis: input.valueSatoshis,
    data: { keys: { privateKeys: { key: privKey } } },
    token: utxoToTokenPrefix(input), // present only for token inputs
  },
}));

const generated = generateTransaction({
  inputs: signedInputs,
  outputs: libauthOutputs, // BIP69-sorted
  locktime: 0,
  version: 2,
});

const txBytes = encodeTransaction(generated.transaction);
const tx_hex = binToHex(txBytes);
```

This is fundamentally different from the "manual signing serialization" path that libauth entity docs emphasize. For P2PKH-only sends the compiler is enough — you don't need `encodeSigningSerializationBCH` + `signMessageHashSchnorr` at all.

**When you DO need manual signing:** CashScript contract inputs (P2SH containing a covenant). Selene's `KeyManagerService.signTemplate` shows the path: `generateSigningSerializationBCH` + `secp256k1.signMessageHashSchnorr` with the sighash flag `allOutputs | utxos | forkId = 0x61`. This is the Phase 2+ path.

## BIP69 + CashTokens output sort

Selene's `bip69SortOutputs` (in TransactionBuilderService.ts) implements the canonical output ordering required by CHIP-2022-02-CashTokens. Sort criteria:

1. value ascending
2. lockingBytecode byte-lex
3. no-tokens < has-tokens
4. fungible token amount ascending
5. no-nft < has-nft
6. NFT capability rank: none < mutable < minting
7. NFT commitment byte-lex
8. category byte-lex

The bch-bot's `lib/sign.mjs` does NOT sort outputs (Phase 1 sends are 1–2 outputs, sort irrelevant). Sort must be added in Phase 2 when multi-output token txs become common.

## Why Selene is the right reference (not moth)

| Aspect | moth | Selene |
|--------|------|--------|
| Source available | Only SKILL.md (public) | Full TypeScript source (BSD-3) |
| Active maintenance | One person, lambda box | Kallisti + contributors, last commit 2026-09-17 |
| CashTokens support | Maintained upstream | Yes, in production wallet |
| Reference quality | Pattern (CLI scripts) | Kernel (TypeScript services) |
| Test coverage | Unknown | `UtxoManagerService.test.ts`, `WalletManagerService.test.ts` |
| Use it for | The CLI script *shape* | The wallet *logic* |

The bch-bot borrows **shape from moth** (thin `.mjs` scripts, single-purpose, BCH_CONFIRM env-gate) and **logic from Selene** (libauth compiler signing, server failover, BIP44 derivation). moth is "what does the API surface look like"; Selene is "how does each piece actually work."

## What this entity does NOT cover

- Selene's UI (React, Redux, Ant Design, Apollo/GraphQL) — not relevant to the bot
- Selene's Capacitor integration (`@capacitor/filesystem`, `SqlJsDatabase`, `capacitor-plugin-simple-encryption`) — the bot uses `node:fs` with mode 0600 instead, plaintext per moth/Selene default
- Selene's BCMR (`@cashscript/cashc`, BCMR registration) — separate service, bot defers to Phase 5
- Cauldron's pool UTXO specifics (the bot will hit this in Phase 4)

## References

- Live repo: <https://gitlab.com/selene.cash/selene-wallet>
- Source files referenced above (verified paths as of 2026-09-17):
  - `package.json` — dep manifest, version `2026.03.08`
  - `src/util/network.ts` — server list, protocol version constants
  - `src/kernel/bch/ElectrumService.ts` — RPC wrapper, error types
  - `src/kernel/bch/TransactionBuilderService.ts` — signing, BIP69 sort
  - `src/kernel/wallet/KeyManagerService.ts` — BIP39/HD, signInputs, signTemplate
  - `src/kernel/wallet/AddressManagerService.ts` — address index management
  - `src/kernel/wallet/UtxoManagerService.ts` — UTXO tracking with token_data