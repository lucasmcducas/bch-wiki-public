---
pageType: synthesis
id: synthesis.security-kb
description: Index + executive summary for the bch-bot security knowledge base. Six reference docs covering BCH UTXO security, wallet key storage, and DEX integration in depth. Original 4 built 2026-09-19; wallet-production audit added 2026-10-01.
sourceUrl: internal/synthesis
---

# bch-bot Security Knowledge Base

> Built by a 4-subagent parallel research team (delegation `deleg_a1c67c78`, 2026-09-19). Each subagent had a focused lens and was given read access to existing wiki content to avoid duplication.
> **Updated 2026-09-19:** new page `entity.omarchy-plugin-marketplace` documents the actual current schema (schemaVersion 1, kinds/entryPoints) — the earlier Omarchy research subagent had a different schema that doesn't match what's live.
>
> **Audience:** future security audits, hardening passes, code reviewers, and the bch-bot user/operator when reasoning about a transaction's safety.

## Documents in this KB

| Doc | What it covers | Author lens |
|---|---|---|
| [`script-and-signing.md`](script-and-signing.md) | Bitcoin Script opcodes (legacy, post-2018 re-enabled, CashTokens introspection), sighash standards (0x41/0x61/FORKID/UTXOS), signature & transaction malleability, low-S requirement, P2SH-32 vs P2S, CashScript covenant patterns | Subagent 1 |
| [`utxo-and-mempool.md`](utxo-and-mempool.md) | Dust thresholds (P2PKH vs P2SH vs NFT-bearing), UTXO selection strategies, fee estimation, mempool policy, double-spend prevention, 0-conf acceptance, the unconfirmed-parent gotcha | Subagent 2 |
| [`wallet-threat-model.md`](wallet-threat-model.md) | BIP-39/32/44 derivation (BCH coin type 145, m/44'/145'/0'), change-chain privacy, key-at-rest, hardware wallet integration, npm supply-chain threats, real-world BCH wallet incidents | Subagent 3 |
| [`cashtokens.md`](cashtokens.md) | Commitment formats (category vs commitment), minting/smelting covenant primitives, transaction replay risks, token indexer trust assumptions, P2SH-32 limitations, Cauldron-specific risks, libauth @cashlab/* patterns | Subagent 4 |
| [`wallet-key-storage.md`](wallet-key-storage.md) | **What four production BCH wallets actually do at rest**, read from source: KDF, cipher, salt and file layout for Cashonize, Selene, Paytaca and Electron Cash — plus the platform shims that silently disable encryption | Subagent (2026-10-01) |
| [`dex-swap-integration.md`](dex-swap-integration.md) | **Integrating a wallet with a CashTokens DEX**: the operator-signing problem, the router's 10 bps fee output and build safety gate, CPMM DAG slippage semantics, the client's pre-signing verification checklist | Subagent (2026-10-01) |
| [`pre-signing-invariant.md`](pre-signing-invariant.md) | **What stands between the router and a signature, and where it is not standing at all**: `send.mjs` and `round-trip.mjs` dead on a missing import, the swap gate that verifies *where* money goes but never *how much*, dry runs that burn change addresses, and the CI shape that lets all of it stay green | Code audit (2026-10-02) |
| [`key-custody-and-oracle.md`](key-custody-and-oracle.md) | **Custody and oracle-layer findings from the live code**: no key zeroisation, unbounded derivation, a fabricated token category, unenforced CashTokens amount bounds, single-node trust with no quorum, a duplicated libauth, and the `encrypt-wallet` backup that undoes its own remediation | Code audit (2026-10-02) |

**Total: ~1,192 lines / ~8,000 words across the original 4 reference docs, plus 2 new pages from the 2026-10-01 production-wallet audit and 2 from the 2026-10-02 live-code audit.**

> **Updated 2026-10-02:** two pages added from a read-only audit of the live
> `~/bch-src` tree at git `d63bda9`. Headline results, in plain terms: `bch-bot send`
> and `bch-bot round-trip` **do not run at all** (they call two functions they never
> imported); the swap gate correctly refuses a redirected output but **passes an
> under-delivery** — the router can promise 36,141 and pay 1 base unit; and a `{}`
> response from `blockchain.transaction.broadcast` is still reported as a success in
> five scripts. Full test suite: **433 assertions, 0 failures** — which is the
> point of the first new page.

## Executive summary (cross-cutting findings)

If you read nothing else, read these:

1. **BCH sighash flags matter more than on BTC.** BCH enforces FORKID since the 2017 split and uses 0x41/0x42/0x43 (all-outputs variants) for normal spends, plus the 0x61 (all-utxos) variant for covenant signing. The bch-bot uses **0x41** correctly for normal P2PKH sends per Selene's template. The 0x61 path (for covenant signing) has a known libauth v3.0.0 bug we're currently routing around with manual signing.

2. **Signature & transaction malleability is still a real concern.** BCH enforces low-S (since Nov 2018 HF), but **transaction-level malleability** (third-party modifying witness/scriptSig without invalidating) is NOT eliminated — third parties can still mutate scripts that produce different encodings. The bot mitigates this by tracking its own txids and not relying on txids received from external sources.

3. **Dust policy is conservative on BCH.** The minimum relay fee (1 sat/byte) means any output below ~546 sat (P2PKH) or ~540 sat (P2SH) is non-economic. The bot's treasury fee module **skips the fee output entirely if the computed fee is below 10 sat** — see `lib/fee.mjs` line ~80.

4. **The "unconfirmed parent" gotcha is a footgun.** Spending a UTXO whose parent tx is still in the mempool fails with `mandatory-script-verify-flag-failed` — misleadingly named, since the real cause is the parent not being on-chain. The bch-bot now exposes `BCH_ALLOW_UNCONFIRMED=yes` as an explicit opt-in for cases where the user has confirmed the parent directly (e.g., it was broadcast by the same wallet).

5. **BCH derivation paths are non-standard.** Most wallets use `m/44'/145'/0'` (BIP-44 standard with BCH coin type 145). Some older wallets used `m/0'/0` (Electron Cash pre-4.0) or `m/44'/0'` (BTC mainnet coin type — wrong for BCH but happens). **The bch-bot uses `m/44'/145'/0'`** (account 0), with a separate change chain at `m/44'/145'/0'/1/i`. This is BIP-44 compliant and matches Selene.

6. **CashTokens security revolves around the indexer.** CashTokens UTXOs carry data in `nftCommitment` and `tokenCategory`; wallets MUST query an indexer (Rostrum) to know which UTXOs hold which tokens. **The indexer is a trust point** — a malicious indexer can lie about balances. The bch-bot uses public Rostrum servers (`cashnode.bch.ninja`, `rostrum.cauldron.quest`); for higher-trust deployments, run your own BCHN node + Fulcrum indexer.

7. **Cauldron pool UTXOs are signed by the pool operator, not the LP.** This means a BCH wallet **cannot** do swaps independently — it must rely on a service (or the user must run the pool themselves). The bch-bot's `lib/cauldron.mjs::buildSwapExecutionTx` builds the user's side of the swap but **does not broadcast** swaps because the pool owner needs to co-sign. **Liquidity provision (LP) is different** — the LP's funds go into a new pool UTXO that the LP themselves controls; this is what `scripts/add-liquidity.mjs --first` does.

8. **PUSD price ≠ $1 always.** Per the existing wiki entity page, PUSD is a stablecoin aiming at $1, but Cauldron's thin liquidity causes deviations. The bch-bot quotes live prices via `lib/cauldron.mjs::getTokenPrice` and surfaces the real rate in swap quotes.

9. **"Encrypted at rest" is a per-platform claim, not a per-wallet one.** Two of the four production wallets audited ship a web implementation whose `encrypt()` returns its input unchanged. Selene's wallet-file export always calls `SimpleEncryption.encrypt` — on web that is a no-op, so the file is plain JSON containing `mnemonic` and `passphrase`. **Check whether a wallet file starts with `{` before assuming it is encrypted.** See [`wallet-key-storage.md`](wallet-key-storage.md).

10. **The most widely used BCH wallet has the weakest KDF.** Electron Cash derives its wallet-file key with `pbkdf2_hmac('sha512', pw, b'', iterations=1024)` — 1024 iterations and an **empty salt**, inherited from Electrum since 2013. Cipher is ECIES with AES-256-CBC + HMAC-SHA256 (authenticated, not AEAD). Inherited-era debt is a recurring theme: Electron Cash's phishing mitigation for server lists was enabled by default and later reverted.

11. **A DEX swap inverts the trust model, and the client's pre-signing checks are the whole defence.** A Cauldron swap spends a pool UTXO committed to the *operator's* key, so the wallet signs a transaction it did not author. The Router discloses which inputs you own, supplies prevouts for all inputs (enabling `SIGHASH_ALL` over the whole tx), and runs a fail-closed safety gate (token conservation, sane miner fee, fee output present, dust). It never broadcasts. None of that replaces local verification — decode the inputs, explain every output, check token `Σ in == Σ out` yourself, and enforce your own slippage floor. See [`dex-swap-integration.md`](dex-swap-integration.md).

12. **No public CVE exists for any production BCH wallet — do not read that as a clean bill of health.** Cashonize maintains an honest `security-considerations.md` and explicitly scopes plaintext seed storage *out* of its vulnerability programme. Documented risk is the dominant failure mode in this ecosystem, not undisclosed vulnerabilities.

> **Correction (2026-10-02).** Executive-summary point 1 and the checklist in
> [`dex-swap-integration.md`](dex-swap-integration.md) state that a token-bearing
> input **must** carry `SIGHASH_UTXOS` (0x61), and that omitting it "is rejected by
> the network with `mandatory-script-verify-flag-failed`". **That overstates the
> spec.** CHIP-2022-02 says the encoded token prefix is included in the signing
> serialization for *all* signing serialization types and *"does not require a
> signing serialization type/flag"*; `SIGHASH_UTXOS` is a security
> **recommendation** (*"wallets **should** enable SIGHASH_UTXOS when
> participating in multi-entity transactions"*), not a validity requirement.
>
> The bch-bot code is correct on both of its paths and does not need changing:
> `signP2pkhTransaction` uses 0x41 for all P2PKH inputs including token-bearing
> ones (`lib/sign.mjs:67`), while `signExternalTransaction` adds the utxos bit for
> router swaps (`lib/sign.mjs:284-286`). So the correction is to the wiki's
> wording, not to the wallet. The `mandatory-script-verify-flag-failed` claim
> should be read as the *actual* failure mode of the preimage bugs in
> [`../references/bch-signing-and-verification.md`](../references/bch-signing-and-verification.md)
> — missing prevouts, a numeric field in a byte position — which is a real and
> separate class.

13. **A green test suite is not evidence a command works.** The 2026-10-02 live-code audit found `send.mjs` and `round-trip.mjs` calling two functions they never imported — so both commands die with `ReferenceError` before touching the network. 433 assertions passed, lint reported clean, and the primary BCH send path could not execute. The cause: `test-send-amount.mjs` re-implements the amount rule instead of importing `send.mjs`, and the lint only checks the *inverse* (imported but unused). See [`pre-signing-invariant.md`](pre-signing-invariant.md).

14. **Ownership checks are not amount checks.** The swap gate byte-compares every output against the addresses we supplied, which correctly catches output redirection. It never compares the *value* of the output paying our own receive address against what the router reported, so a router that promises 36,141 and pays 1 base unit passes every gate. Verified by running the project's own gate against hand-built transactions. See [`pre-signing-invariant.md`](pre-signing-invariant.md) §2.

15. **The key-storage posture is sound; the oracle layer is not.** The mainnet wallet directory is `0700` with both files `0600`, and no seed or private key can reach a log or an error message. Against that: one Electrum node is the sole source of truth with no cross-check, and a `{}` response from `blockchain.transaction.broadcast` is still reported as `"broadcast": true` in five scripts — a documented bug class that was fixed in `lib/router.mjs` and never applied to its siblings. See [`key-custody-and-oracle.md`](key-custody-and-oracle.md).

## How to use this KB

- **Before a security audit:** read all six docs end-to-end (~45 min). Each doc cites primary sources (BIPs, BCHN release notes, CashScript specs, wallet source files) so you can verify claims.
- **When hardening bch-bot:** the `wallet-threat-model.md` doc maps specific threat classes to specific code paths in `lib/wallet.mjs`, `lib/sign.mjs`, and `lib/network.mjs`.
- **When comparing your wallet to production wallets:** `wallet-key-storage.md` has the KDF/cipher/salt table for Cashonize, Selene, Paytaca and Electron Cash, read from source.
- **When integrating any DEX or swap UI:** `dex-swap-integration.md` has the pre-signing verification checklist and the slippage semantics.
- **When debugging a stuck transaction:** start with `utxo-and-mempool.md` (dust policy + unconfirmed-parent gotcha).
- **When adding a new CashTokens operation:** start with `cashtokens.md` (commitment format + indexer trust).
- **When changing the sighash flag or signing template:** start with `script-and-signing.md` (0x41 vs 0x61).
- **When auditing or hardening the bot's own code (2026-10-02):** start with `pre-signing-invariant.md` — it covers the dead commands, the unverified receive amount, and the dry-run state mutation that the other six docs do not. `key-custody-and-oracle.md` covers custody, the untrusted-node layer, and supply chain.

## What this KB is NOT

- **Not a security audit.** It's a knowledge base — a foundation. Real audit work needs to be done against the live code with these references as background.
- **Not exhaustive.** BCH has been actively developed since 2017; this KB captures the state as of Sept 2026. BCHN forks after this date may change the consensus rules; check `sources/bchn-release-notes.md` for recent changes.
- **Not a substitute for primary sources.** Every claim has a citation. When in doubt, read the primary source.

## Provenance

| Field | Value |
|---|---|
| Built by | 4 subagents in `deleg_a1c67c78` |
| Date | 2026-09-19 |
| Dispatched by | Hermes (Jav) |
| Existing wiki content respected | entities/libauth.md, entities/cashscript.md, entities/selene-wallet.md, references/bch-zero-conf-security.md, references/bch-economics-primer.md |
| Total lines | 1,192 |
| Total words | ~8,000 |
