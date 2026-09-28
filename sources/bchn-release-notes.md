<!-- openclaw:wiki:raw-source -->
---
pageType: source
sourceType: release-notes
sourceUrl: https://docs.bitcoincashnode.org/doc/release-notes/
title: Bitcoin Cash Node release notes (v25, v28, v29 series)
accessedAt: 2026-09-14
description: Release notes for BCHN versions relevant to CashTokens, P2SH-32, May 2025 upgrade (VM limits + BigInt), and May 2026 upgrade (P2S + loops + functions + bitwise).
---

# Source: Bitcoin Cash Node release notes (v25 / v28 / v29 series)

**Source**: https://docs.bitcoincashnode.org/doc/release-notes/
**Accessed**: 2026-09-14

The release notes are the authoritative reference for what changed RPC-by-RPC and consensus-rule-by-consensus-rule. Below is the distilled content relevant to smart-contract development and our BCH LLM corpus.

## v29.1.0 (2026-07-28) — minor release

### Network changes
None.

### Added functionality
- New `-rpcwaittimeout` for `bitcoin-cli`.
- Default `rpcthreads` raised 4→16, `rpcworkqueue` 16→64.
- `getmempoolinfo` now returns `total_fee`.
- `testmempoolaccept` returns more info.
- `gettxout` has new 4th arg `patterns` (boolean) → emits `byteCodePatterns` object.
- `getnetworkinfo`, `getpeerinfo`, `getnodeaddresses` return human-readable network service flags.
- `getnodeaddresses` returns a "network" field.
- `getrpcinfo` returns `logpath`.
- `getchaintxstats` returns `window_final_block_height`.

### New REST endpoints
- `/rest/spenttxouts/<BLOCK-HASH>.<bin|hex|json>`
- `/rest/spenttxouts/withpatterns/<BLOCK-HASH>.<bin|hex|json>`

### Limitations (verbatim — note this is also in v28 and v29.0.0)

> 1. CashToken support is low-level at this stage. The wallet application does not yet keep track of the user's tokens. Tokens are only manageable via RPC commands currently. They only persist through the UTXO database and block database at this point. There are existing RPC commands to list and filter for tokens in the UTXO set. RPC raw transaction handling commands have been extended to allow creation (and sending) of token transactions. Interested users are advised to consult the functional test in `test/functional/bchn-rpc-tokens.py` for examples on token transaction construction and listing. **Future releases will aim to extend the RPC API with more convenient ways to create and spend tokens, as well as upgrading the wallet storage and indexing subsystems to persistently store data about tokens of interest to the user. Later we expect to add GUI wallet management of Cash Tokens.**
>
> 2. Transactions with SIGHASH_UTXO are not covered by DSProofs at present.
>
> 3. P2SH-32 is not used by default in the wallet (regular P2SH-20 remains the default wherever P2SH is treated).
>
> 4. The markup of Double Spend Proof events in the wallet does not survive a restart of the wallet, as the information is not persisted to the wallet.
>
> 5. The ABLA algorithm for BCH is currently temporarily set to cap the max block size at 2GB.

## v29.0.0 (2026-01-09) — May 15, 2026 Network Upgrade

### Consensus CHIPs activated
- **CHIP-2024-12 P2S: Pay to Script** — pay to any bytecode directly (not via P2SH wrap). New template lock for covenant-heavy contracts.
- **CHIP-2021-05-loops: Bounded Looping Operations** — bounded loops in Script (`OP_LOOP`, `OP_BREAK` etc.).
- **CHIP-2025-05 Functions: Function Definition and Invocation Operations** — composable Script functions.
- **CHIP-2025-05 Bitwise: Re-Enable Bitwise Operations** — `OP_AND`, `OP_OR`, `OP_XOR`, `OP_INVERT` and friends re-enabled.

### Added functionality
- New `-coinstatsindex` (off by default) + extended `gettxoutsetinfo` with `hash_serialized_3`, ECMH, MuHash3072.
- New `patterns` flag for `getblock` and `getrawtransaction` → `byteCodePattern` per script.
- REST endpoints `/block/withpatterns/<HASH>`, `/tx/withpatterns/<HASH>`.
- `getnetworkinfo`/`getpeerinfo` keys.
- `-peerratelimit` for per-peer bandwidth limiting.

### Modified
- `gettxoutsetinfo` — uses `hash_serialized_3` by default; can now query historical block stats via `coinstatsindex`.
- `getblockchaininfo` — new `upgrade_status` field.

## v28.0.0 (2024-11-30) — May 15, 2025 Network Upgrade

### Consensus CHIPs activated
- **CHIP-2021-05 VM Limits: Targeted Virtual Machine Limits** — expanded VM execution limits.
- **CHIP-2024-07 BigInt: High-Precision Arithmetic for Bitcoin Cash** — arbitrary-precision integers up to 80,000 bits.

### User interface changes
- `getrawtransaction` with `verbosity=2` now returns `tokenData` and `scriptPubKey` per output.

### New RPC methods
- `getindexinfo` — returns running indices and their sync status.

### Limitations (verbatim)

> CashToken support is low-level at this stage. The wallet application does not yet keep track of the user's tokens. Tokens are only manageable via RPC commands currently. They only persist through the UTXO database and block database at this point. There are existing RPC commands to list and filter for tokens in the UTXO set. RPC raw transaction handling commands have been extended to allow creation (and sending) of token transactions. Interested users are advised to consult the functional test in `test/functional/bchn-rpc-tokens.py` for examples on token transaction construction and listing. Future releases will aim to extend the RPC API with more convenient ways to create and spend tokens, as well as upgrading the wallet storage and indexing subsystems to persistently store data about tokens of interest to the user. Later we expect to add GUI wallet management of Cash Tokens.

## v25.0.0 (2023-05-15) — CashTokens activation

### Consensus CHIPs activated
- **CHIP-2021-01 Restrict Transaction Version v1.0**
- **CHIP-2021-01 Minimum Transaction Size v0.4**
- **CHIP-2022-02 CashTokens v2.2.1**
- **CHIP-2022-05 P2SH32 v1.5.1**

### Network changes
- New `chipnet` test network (-chipnet) — token-aware testnet, default ports 48333/48332/48334.

### Modified functionality — CashTokens-aware RPCs (the canonical list)

The following RPC methods were extended to transport or accept CashTokens information. **This is the foundation of every BCH token wallet built today — there is no higher-level wrapper in BCHN.**

| RPC | CashTokens-related extension |
|---|---|
| `gettxout` | `tokenData` JSON object present if output contains token data |
| `getrawtransaction` (verbosity ≥1) | Returns token info on outputs |
| `decoderawtransaction` | Returns token info on outputs |
| `decodepsbt` | `input.utxo` extended with optional `tokenData` |
| `listunspent` | Two new boolean options controlling the listed results (filter tokens) |
| `signrawtransactionwithkey` / `signrawtransactionwithwallet` | Accept `tokenData` on prevout objects |
| `createrawtransaction` | Accepts `data` array for token mint/burn |
| `scantxoutset` | Token filter options |
| `getdsproofscore` | DSProof scoring RPC |

### New RPC methods
- `getindexinfo` (later in v28.0.0; v25 also had additions)

### New sighash type
- `SIGHASH_UTXO` (0x20) — introduced for CashTokens spec.

## Why this source matters for our LLM

1. **Verified-correct transaction construction examples.** Every `getrawtransaction` / `decoderawtransaction` / `signrawtransactionwithwallet` example in these release notes is an authoritative reference. Use them as SFT seed data — they are guaranteed to reflect what BCHN actually accepts.
2. **The "limitations" section is load-bearing.** It tells you what the LLM must NOT promise: no GUI token management, no built-in balance tracking, no high-level wallet RPC for tokens. A model that hallucinates `gettokenbalance` is wrong.
3. **`bchn-rpc-tokens.py`** (mentioned in every release) is a 700+ line functional test that constructs every kind of token transaction. Top-priority training data.

## Related

- [Bitcoin Cash Node entity](entities/bchn-node.md)
- [CashTokens concept](concepts/cash-tokens.md)
- [Cauldron DEX](entities/cauldron-dex.md)
