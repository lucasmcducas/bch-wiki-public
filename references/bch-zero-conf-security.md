---
pageType: reference
id: reference.bch-zero-conf-security
description: How 0-confirmation transactions are protected (and not protected) on Bitcoin Cash mainnet in 2026, and whether the bot can safely spend an unconfirmed UTXO.
sourceUrl: https://upgradespecs.bitcoincashnode.org/dsproof/
---

# BCH Zero-Confirmation Security

*Research note for the bch-bot. Current as of 2026-09-17.*

## 1. The premise correction: BCH ≠ eCash (XEC)

The parent brief says "BCH added Avalanche pre-consensus around 2024." **This is wrong.** Avalanche is live on **eCash (XEC)**, not on Bitcoin Cash (BCH). Bitcoin Cash Node (BCHN) and eCash (Bitcoin ABC) split in November 2020 after the IFP controversy; they are now separate projects with separate roadmaps.

- **Avalanche Post-Consensus on eCash:** activated September 14, 2022 (Bitcoin ABC v0.26.2+, opt-in via `avalanche=1`). Finalizes blocks pre-conflict.
- **Avalanche Pre-Consensus on eCash:** activated November 15, 2025 at XEC block height 923,347. Finalizes individual transactions before they're mined — reduces confirmation time "from 10 minutes to under 3 seconds" per eCash team.
- **Bitcoin Cash (BCHN):** no Avalanche. As of BCHN v29.1.0 (current), 0-conf protection on BCH mainnet comes from **(a)** the absence of RBF and **(b)** Double Spend Proofs (DSProofs).

## 2. What 0-conf on BCH actually looks like

A transaction is **0-conf** from the moment it leaves the sender's wallet until it is buried under ≥1 block. The receiver sees it in their mempool. Mempool acceptance follows the standard Bitcoin rule: when two transactions spend the same UTXO, nodes keep the first one they see (`first-seen` policy) and drop the second. There is no on-chain penalty for broadcasting the double-spend.

Three properties make BCH 0-conf **materially safer** than BTC 0-conf:

1. **No RBF.** BIP125 (opt-in replace-by-fee) was intentionally removed from BCH at the 2017 fork. A BCH sender cannot broadcast a "replacement" tx spending the same inputs at a higher fee, which is the dominant BTC double-spend vector.
2. **Larger blocks + faster block time.** BCH's block size is unbounded (capped only by ABLA since 2024) and blocks average ~10 min. Propagation is fast enough that a well-connected sender's tx reaches the economic majority of nodes within seconds.
3. **Double Spend Proofs (DSProofs).** Spec by Tom Zander / imaginary_username (2020); live in BCHN since v23.0.0 (October 2022); default-on. When a BCHN full node sees two signed transactions spending the same outpoint, it constructs a compact cryptographic proof (the two `tx_sig_preimage`s plus their signatures) and gossip it as INV type `0x94a0` / `dsproof-beta`. SPV wallets and merchant POS apps can subscribe to DSProofs and learn about the conflict within seconds.

DSProofs cover "protected transactions" — `SIGHASH_ALL | FORKID` without `ANYONECANPAY`, P2PKH or P2SH-multisig with unique pubkeys. This is the overwhelming majority of what BCH wallets produce today (Selene uses libauth + Schnorr, fits the protected class).

**Critical gap:** DSProofs do **not** cover miner-assisted double-spends. Tom Zander's own writeup says: *"we don't protect against miner-assisted double spends."* If the sender controls or bribes a BCH miner, they can mine a conflicting tx themselves and DSProofs won't save you. The attack requires hash power.

## 3. Avalanche is not the BCH story

The eCash project (separate from BCH since 2020) is the one running Avalanche. eCash activated Pre-Consensus in November 2025, giving it sub-3-second transaction finality. None of that machinery exists on BCH mainnet. **Any wiki page that confuses the two is wrong; any source from the eCash team (e.cash, avalanche.cash, Bitcoin ABC blog) describes XEC, not BCH.**

## 4. Risk assessment for the bot's 0.00858627 BCH spend

The bot just received an unconfirmed UTXO (txid `25d771fb…0ac2`) and wants to immediately spend 1000 sats to a fresh address as a "round-trip" smoke test. Risk analysis:

| Property | Value | Implication |
|----------|-------|-------------|
| Amount at risk | 0.00858627 BCH (~1–5 USD at 2026 prices) | Below any rational double-spend cost |
| Sender | Self (the bot's own previous tx) | No adversarial sender |
| Destination | Fresh address (bot-controlled) | Not exposed to a third-party merchant |
| Time at risk | Seconds (mempool only) | Doubly-spend window is the block interval |
| Network | BCH mainnet with DSProof gossip | First-seen + DSProof will catch a conflict |
| Miner collusion needed | Yes, to override both first-seen and a DSProof | Cost > value for 1000 sats |

**Verdict: safe to spend at 0-conf for this specific use case.** The double-spend would require the sender itself to attempt fraud against itself, which is irrational. The 1000-sat round-trip is a valid smoke test of mempool acceptance + propagation + dust-absorption logic; it is *not* a stress test of BCH finality, and it shouldn't be mistaken for one. If it succeeds, you have confirmed: (i) the wallet can find the new UTXO via the indexer, (ii) it can sign a tx spending that UTXO, (iii) the tx propagates, (iv) the resulting child output survives the dust threshold or is correctly absorbed into fee.

What you have **not** tested: miner reorg survival (would need a deeper fork), Avalanche finality (not applicable on BCH), or DSProof detection (would need to deliberately double-spend).

## 5. Recommended waiting policy

| Value at risk | Min wait before trusting as settled |
|---|---|
| < $1, dust, self-to-self | **0-conf OK** (current bot use case) |
| $1–$100 (POS coffee) | 0-conf + DSProof monitor; or 1-conf |
| $100–$1,000 | 1-conf (~10 min) |
| $1,000–$10,000 | 3-conf (~30 min) |
| > $10,000 (exchange deposit, large vendor) | 6–10-conf; consider a wait-list + DSProof rejection logic |

BCH reorgs > 1 block are extremely rare (the deep-reorg protection in BCHN rejects anything > 10 blocks deep). 6-conf is the historical "exchange standard" (BitPay default for BCH deposits), roughly 1 hour.

## 6. Contrarian / dissent view

Not all of the BCH community agrees 0-conf is safe. The most cited dissent is **Max Hastings** on bitcoincashresearch.org (Oct 2021, "Why some services can not adopt 0-confirmation transactions"): a service that *accepts 0-conf deposits* and *lets the user withdraw* before confirmation is trivially exploitable. An attacker with even 0.1% of hash power (~1/1000 of BCH hashrate) can win a double-spend against a 0-conf-accepting service in statistically ~1 week by repeatedly trying. The attacker loses nothing on failed attempts and earns the block reward on successful ones. **DSProofs do not solve this**, because by the time the merchant sees the proof and freezes the withdrawal, the attacker has already walked away with the funds from the destination exchange. The mitigation the BCH research community proposed is **CHIP-2021-08 Zero-Confirmation Escrows (ZCEs)** — a covenant-based construction that makes 0-conf-recipient safe even against miner-assisted double-spends. ZCEs are still in design, not deployed.

The **practical BCH community consensus** (reddit r/btc, BCHN developer statements) is closer to: *0-conf is fine for in-person retail under $100 where the merchant can see the customer and refuse service on DSProof alert. 0-conf is **not** safe for unattended deposits where the sender can walk away after the funds are credited.*

## 7. Sources

- BCHN upgrades / DSProof spec: https://upgradespecs.bitcoincashnode.org/dsproof/
- BCHN DSProof implementation notes: https://docs.bitcoincashnode.org/doc/dsproof-implementation-notes/
- DSProof message spec (imaginary_username): https://github.com/imaginaryusername/specs_n_stuff/blob/master/dsproof/dsproof.md
- BCHN v23.0.0 release notes (DSProof default-on): https://docs.bitcoincashnode.org/doc/release-notes/release-notes-23.0.0/
- BCHN v29.1.0 (current): https://github.com/bitcoin-cash-node/bitcoin-cash-node/releases
- Max Hastings, "Why some services can not adopt 0-confirmation transactions" (dissenting view): https://bitcoincashresearch.org/t/why-some-services-can-not-adopt-0-confirmation-transactions/592
- Tom Zander on DSProof limits: https://read.cash/@TomZ/making-regular-payments-more-secure-with-dsproof-e0ceea3d
- eCash Avalanche pre-consensus announcement (NOT BCH): https://e.cash/blog/sneak-peek-avalanche-pre-consensus-on-ecash
- eCash Avalanche mainnet activation (NOT BCH): https://www.bitcoinabc.org/2022-09-29-avalanche-post-consensus/
- BCH / eCash split history: https://en.wikipedia.org/wiki/Bitcoin_Cash
- BCH Reddit 0-conf threads: https://www.reddit.com/r/btc/comments/1bd3u5y/how_many_block_confirmations_are_needed_for/, https://www.reddit.com/r/btc/comments/vxr3qf/explaining_0_conf_transactions/
- moth bch-wallet skill (this project's spend path): https://dev.selene.technology/~moth/skills/ (skill: `bch-wallet`)

## 8. Operational rule for the bot

**Spend 0-conf only when all four are true:** (i) the incoming tx is self-originated OR the sender's identity/reputation outweighs the at-risk amount, (ii) the value is < the cost of a rational double-spend attack (including any bribe to a miner), (iii) the bot has a DSProof monitor running (subscribe via BCHN `getdsprooflist` RPC or equivalent Electrum/Chronik hook), and (iv) the destination is bot-controlled or accepts reversals. For the 1000-sat round-trip specifically: all four are satisfied, proceed.
