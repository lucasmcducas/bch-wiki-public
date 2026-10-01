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

**Total: ~1,192 lines / ~8,000 words across the original 4 reference docs, plus 2 new pages from the 2026-10-01 production-wallet audit.**

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

## How to use this KB

- **Before a security audit:** read all six docs end-to-end (~45 min). Each doc cites primary sources (BIPs, BCHN release notes, CashScript specs, wallet source files) so you can verify claims.
- **When hardening bch-bot:** the `wallet-threat-model.md` doc maps specific threat classes to specific code paths in `lib/wallet.mjs`, `lib/sign.mjs`, and `lib/network.mjs`.
- **When comparing your wallet to production wallets:** `wallet-key-storage.md` has the KDF/cipher/salt table for Cashonize, Selene, Paytaca and Electron Cash, read from source.
- **When integrating any DEX or swap UI:** `dex-swap-integration.md` has the pre-signing verification checklist and the slippage semantics.
- **When debugging a stuck transaction:** start with `utxo-and-mempool.md` (dust policy + unconfirmed-parent gotcha).
- **When adding a new CashTokens operation:** start with `cashtokens.md` (commitment format + indexer trust).
- **When changing the sighash flag or signing template:** start with `script-and-signing.md` (0x41 vs 0x61).

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
