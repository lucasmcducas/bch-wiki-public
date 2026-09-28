---
pageType: reference
id: reference.security.utxo-and-mempool
description: UTXO selection, dust policy, fee estimation, mempool policy, and double-spend prevention on Bitcoin Cash — what a BCH wallet bot needs to know to construct, broadcast, and reason about transactions.
sourceUrl: https://docs.bitcoincashnode.org/doc/release-notes/release-notes-27.0.0/
---

# BCH UTXO Selection, Dust Policy, Mempool & Double-Spend Prevention

*Reference for the bch-bot. Complements `references/bch-zero-conf-security.md` (which covers DSProofs, FSS, and the no-RBF question at the *acceptor* side); this doc covers the *sender/constructor* side: dust math, fee math, coin selection, and the relay policy that decides whether a transaction even reaches a miner. Current as of 2026-09-19.*

## 1. Dust threshold: what "dust" means on BCH

### 1.1 The formula

BCH uses the same dust rule as Bitcoin Core, inherited from `src/policy/policy.cpp::GetDustThreshold`. An output is **dust** if its value (in satoshis) is less than the cost of spending it at the dust-relay fee rate:

```
dust_threshold = (3 × input_bytes + output_bytes + 41) × dust_relay_fee_per_byte
```

For a 34-byte P2PKH output (the canonical case: `OP_DUP OP_HASH160 <20-byte hash> OP_EQUALVERIFY OP_CHECKSIG`) with the default dust-relay fee of 1000 sat/kvB = 1 sat/byte:

```
dust_threshold = (3 × 148 + 34 + 41) × 1 = 519 satoshis
```

…which Bitcoin Core and BCHN both round up to **546 satoshis**. This is the "546" number you see everywhere.[^1][^3]

### 1.2 Per-output-type variation

The threshold *changes per script type* because the input-byte estimate (148 for legacy P2PKH, ~107 for P2WPKH, ~297 for P2SH-multisig) and the output-byte cost change. For the **BCH ecosystem in 2026** the cases that matter are:

| Output type | Output bytes | Approx dust at 1 sat/B |
|---|---|---|
| P2PKH (legacy, cashaddr `q...`/`p...`) | 34 | **546** |
| P2SH-multisig (cashaddr `q...`/`p...` with script hash) | 32 + N×~72 | ~540 + ~210·N — substantially higher than P2PKH |
| CashTokens NFT (carries `nft` prefix) | adds ~40 bytes for token prefix + commitment | **~1000+ sats** in practice; libauth's `getDustThreshold` computes this dynamically |
| OP_RETURN (data carrier) | variable | exempt — no spendable output, so no dust check |

Sources: Bitcoin Core 0.14+ dust formula as documented in the Stack Exchange thread on `policy.cpp`[^1] and the Bitcoin Cash protocol reference's "Network-Level Validation Rules" page, which explicitly states "a threshold of 546 satoshis is used" in the simplest case.[^2]

### 1.3 CashTokens dust

For CashTokens outputs, the **dust threshold is calculated dynamically** by the library / node from the actual byte cost of the output. The bch-bot uses `libauth`'s `getDustThreshold(output, dustRelayFee)` — see `lib/tokens.mjs:67`:

```javascript
const DUST_RELAY_FEE = 1000n; // 1 sat/byte (BCH standard)
...
const dustThreshold = getDustThreshold(output, DUST_RELAY_FEE);
if (satsAmount < dustThreshold) {
  output.valueSatoshis = dustThreshold;
}
```

The default `satsAmount` for FT/NFT outputs in the bot is **1000 sats** (`createTokenOutput({ satsAmount: 1000n })`) — chosen to be safely above the dust floor for any token-bearing P2PKH output, including those carrying an NFT commitment.[^bot-tokens]

### 1.4 Why dust matters

A sub-dust output cannot be relayed by honest nodes (their mempool will reject the parent tx). Three operational consequences:

1. **You can't *send* sub-dust.** A wallet constructing a tx with an output below the threshold will see it bounced by the first node that validates it.
2. **Change can be silently absorbed into the fee.** The bch-bot's `scripts/send.mjs:81–88` implements the standard wallet behaviour: if the change output would be sub-dust, drop the change output entirely; the difference becomes part of the fee. The bot logs `"change N sats below dust; absorbed into fee"`.[^bot-send]
3. **Dust UTXOs become unspendable for practical purposes.** Even if a node *will* relay a spend of a dust UTXO (some wallets allow it via `-acceptnonstdoutputs` or by overriding policy), the cost of spending it (3 × input_bytes × fee_rate) exceeds its value at any reasonable fee rate. This is the motivation behind `scripts/sweep.mjs`[^bot-sweep], which consolidates sub-5000-sat UTXOs into a single fresh change output. The bot sets `MIN_REMAINING_AFTER_FEE_SATS = 500` and refuses to sweep if the result would leave a smaller dust problem.

## 2. UTXO selection strategies

### 2.1 The algorithms, briefly

Wallets have five families of coin-selection algorithm to choose from. Most modern BCH wallets (Selene, Electron Cash, Cashonize) follow the Bitcoin Core playbook here:

| Algorithm | How it works | Privacy | Fee efficiency | Use case |
|---|---|---|---|---|
| **Largest-first** | Sort UTXOs by value descending; greedily add until target + fee met | Worst — links large + small spends of the same owner | Best — fewest inputs = lowest fee | Bot's default (`send.mjs:39`)[^bot-send] |
| **Branch-and-Bound (BnB, "Murch's algorithm")** | Exhaustive bounded search of input combinations; picks the one that minimises *waste* (excess fee + long-term fee cost of change) | Better — uses mid-sized UTXOs that don't cluster with the wallet's other large spends | Best at low fee rates — finds exact-match no-change solutions | Bitcoin Core 0.17+ default[^bnb-ref] |
| **Knapsack** | Random-subset-of-UTXOs random walk until target met; biased toward more inputs | Best against chain-analysis — looks like a different wallet each time | Worst — over-shoots and pays change | Bitcoin Core pre-0.17 default; still used as fallback[^knapsack-ref] |
| **Single Random Draw** | Pick UTXOs uniformly at random until target met | Good — diverse input selection | Variable | Bitcoin Core fallback (post-0.17)[^srd-ref] |
| **FIFO / oldest-first** | Prefer the UTXOs with the earliest confirmation time | Worst — long history creates rich chain-analysis heuristics | Variable | Used by some custodial systems, rarely by self-custody wallets |

The **waste metric** (introduced with BnB and generalised in Bitcoin Core PR #22009) is the shared scoring function across algorithms: `waste = fees_paid + change_cost + excess_to_fee`. The "best" solution is the one with the lowest waste.[^waste-ref]

### 2.2 What the bch-bot does

The bot uses **largest-first** as a deliberate simplicity choice, with two important caveats:

```javascript
// lib/tokens.mjs:164 — BCH input selection
const bchOnly = allUtxos
  .filter((u) => !u.token_data)            // ← never spend a token-bearing UTXO into fee
  .sort((a, b) => b.value - a.value);      // ← largest first
const bchSelected = [];
let bchTotal = 0n;
for (const u of bchOnly) {
  bchSelected.push(u);
  bchTotal += BigInt(u.value);
  if (bchTotal >= bchRequired + 1000n) break;  // ← 1000-sat safety margin over target
}
```

The `+1000n` margin in the break condition is critical: at the default fee rate of 1 sat/byte and a worst-case 200-byte input, the actual fee is ~200 sats per input. The 1000-sat buffer covers (i) fee variance from extra inputs, (ii) token prefix bytes if an input accidentally carries tokens, (iii) signing-serialization overhead.[^bot-tokens]

The trade-off the bot accepts: largest-first is the worst option for privacy. **For an automated treasury bot doing scripted sends, privacy is a secondary concern; fee efficiency and predictability dominate.** A human-operated wallet (Selene, Electron Cash) should run BnB by default.

### 2.3 Token-aware selection

`lib/tokens.mjs::selectInputsForTokenSend` runs two passes:

1. **FT pass:** `selectTokenUtxos(allUtxos, category, targetAmount)` — filter to UTXOs of the target category, sort by FT amount descending, sum until target met. *Only the FT inputs carry the token; spending them in a "wrong" tx would burn the tokens.*[^bot-tokens]
2. **BCH pass:** as above, but the target is `bchRequired = output_sats + estimated_fee`. BCH-only UTXOs are preferred over token-bearing ones to avoid accidental token burn into fee.

The pattern is borrowed from Selene's `TransactionBuilderService` (verified against the Selene Wallet repository structure; the bot's comments cite `src/util/normalize.ts` and `src/kernel/bch/TransactionBuilderService.ts`).[^bot-tokens]

## 3. Fee estimation

### 3.1 The 1 sat/byte default

BCHN's default `-minrelaytxfee` is **1000 satoshis/kvB = 1 sat/byte** — the same value as legacy Bitcoin Core (Bitcoin Core has since lowered this to 0.1 sat/byte in PR #33106, but BCHN has not followed).[^minrelay-ref][^reddit-relay] The bot's constant `FEE_RATE_SATS_PER_BYTE = 1.0` (`scripts/send.mjs:15` and `scripts/round-trip.mjs`) is hard-coded to this default.

Practical implication: a typical 1-input 2-output P2PKH tx weighs ~200 + 34 + 34 = **~268 bytes** (plus ~10 bytes overhead), so the minimum fee is **~268 sats** at the relay floor. Real-world fee bids cluster between 1 and 2 sat/byte because:

- Miners earn almost nothing from 1 sat/byte during normal congestion
- Wallet defaults (Selene, Electron Cash, Cashonize) use 1 sat/byte as the safe minimum
- Confirmed-time variance at 1 sat/byte is small on BCH because block space is currently abundant (ABLA cap is 2 GB, typical blocks are 100 KB–5 MB)[^abla-ref]

### 3.2 When fees spike

BCH fees spike when (a) mempool fills faster than ABLA can grow the block cap (currently bounded at 2 GB, a "temporary" p2p limit), or (b) a CashTokens minting wave floods the mempool with low-fee dust. Practical observations from block explorers:

- **Normal mainnet:** 1–2 sat/byte confirms in 1–3 blocks (~10–30 min). Median confirmation: 1 block.
- **CashTokens mint event:** mempool can spike to 50–200 MB; fee rates observed 2–5 sat/byte for inclusion in the next block. The spike typically clears in <2 hours as the minting burst ends.
- **Stress (rare):** 5–10 sat/byte sustained for hours during major protocol upgrades.

### 3.3 What the bot does

The bot's fee estimation is **static** (1 sat/byte) rather than dynamic. This is fine for the bot's current workload (low-frequency treasury moves, small round-trip smoke tests). For higher-value sends or during known fee spikes, the operator should override the constant or pass `--fee-rate` if added. **Caveat:** the bot has no fee-estimation API client (no Electrum `estimatefee` call); it relies on the constant being current.

### 3.4 RBF / CPFP note (the asymmetry)

BCH has **no RBF** (see §4). The fallback is **CPFP — Child-Pays-For-Parent**. If a tx stalls because its fee is too low, the *recipient* can spend one of its outputs with an elevated fee; miners evaluate the package (parent + child) at the average fee rate and include both.[^cpfp-ref]

The bot does not currently implement CPFP acceleration. For a stuck tx, the options are:

1. Wait for fee pressure to subside and the tx to confirm naturally.
2. Create a "booster" tx that spends one of the stuck outputs to the bot's own address with a high fee (classic CPFP).
3. Double-spend the stuck tx with a higher-fee replacement. **This is not RBF** — it's just a new tx spending the same inputs. It works on BCH because there is no first-seen protection *between competing 1-conf transactions* until the first-seen rule (§4) decides. In practice, the original tx will still be in the mempool and your "booster" will be rejected unless it has materially different outputs. The reliable CPFP path is option 2, not option 3.

## 4. Mempool policy on BCH

### 4.1 First-seen rule

BCH enforces the standard Bitcoin first-seen mempool rule: when a node receives two transactions spending the same outpoint, it keeps the first and rejects the second.[^fss-ref] There is **no opt-in RBF flag, no full-RBF, no replace-by-fee at all** — BIP125 was deliberately removed at the 2017 fork because BCH's design prioritises 0-conf safety for in-person retail.[^bch-zero-conf]

This is the single most important policy difference from BTC for a wallet bot. On BTC, a sender can:

- Mark the tx as RBF (`signals: { rbf: true }` in PSBT)
- Broadcast a replacement with a higher fee rate, replacing the original in mempools
- Get the replacement confirmed, voiding the original

On BCH this attack vector is closed by design.

### 4.2 Mempool size

Bitcoin Core defaults to 300 MB mempool (`DEFAULT_MAX_MEMPOOL_SIZE = 300`). BCHN inherits this default. When the mempool fills, the node evicts transactions with the lowest descendant feerate first and raises the dynamic minimum feerate, rejecting new transactions below the new floor.[^mempool-size]

BCH rarely fills its mempool to eviction because of ABLA (block space is currently capped at 2 GB per block). The bot has never observed eviction on chipnet or mainnet under normal load.

### 4.3 Max standard tx size

BCHN's `MAX_STANDARD_TX_SIZE` is **100,000 vbytes** by default — inherited from Bitcoin Core pre-SegWit. The exact constant in BCHN's `src/policy/policy.h` is `MAX_STANDARD_TX_SIZE = 100000`. **This is a *standardness* policy, not consensus; miners can include larger txs.** ABLA's 2 GB block cap makes the limit largely theoretical for BCH today — a "huge" CashTokens mint batch might push 100 KB but rarely anywhere near the cap.[^max-tx-gap]

**Gap:** I have not verified the BCHN-specific value of `MAX_STANDARD_TX_SIZE` against the v29.1.0 source in this writeup. The 100,000 vbyte value is the standard Bitcoin Core inheritance and would surprise no one if BCHN changed it.

### 4.4 What nodes do *not* check at mempool time

Some things that *don't* trigger mempool rejection but *do* trigger block-rejection:

- Script validity in deeper detail (P2SH redeemScript validation is lazy)
- Token prefix correctness at the libauth level (a malformed `token:` field will be accepted into the mempool if it parses, then rejected by miners at consensus)
- CashScript contract execution (always deferred to block validation)

This is why a tx that "broke at consensus" can still appear in mempool explorers for a few seconds before disappearing. For the bot, the operational rule is: **if `broadcast` returns success, the tx will reach miners; whether miners include it depends on fee and policy, not on script correctness.**

## 5. Double-spend prevention

The "is BCH 0-conf safe?" question is covered in detail in `references/bch-zero-conf-security.md`. The summary, from the *constructor* (sender) side:

- **First-seen rule** prevents a simple rebroadcast of an alternative tx from being accepted by the same node.
- **Double Spend Proofs (DSProofs)** — when a node sees two signed txs spending the same outpoint, it constructs a compact proof of the conflict and gossips it (INV `0x94a0` / `dsproof-beta`). Receivers subscribed to DSProofs learn about the conflict within seconds.[^bch-zero-conf]
- **Miner-assisted double-spends are still possible.** A sender who controls (or bribes) a miner can mine a conflicting tx themselves. DSProofs do *not* prevent this; they only detect it. **For unattended deposits >$100, wait for ≥1 confirmation.**[^bch-zero-conf]

### 5.1 What the bot should do as a *sender*

When the bot is the sender (the common case for treasury moves), double-spend prevention is largely a non-issue — the bot only signs one tx per intent. The relevant rules:

1. **Never sign two txs spending the same inputs at the same time.** If you need to "cancel" a stuck tx, do it via CPFP (§3.4 option 2), not by broadcasting an alternative. The alternative will be rejected by the first-seen rule anyway (since the original is already in mempool), but you waste a signature and possibly leak your intent.
2. **Always broadcast via well-connected nodes.** The bot uses `cashnode.bch.ninja:50004` first in its failover list (Selene's "canonical" server)[^bot-network] — this maximises the chance of the tx reaching the economic majority of nodes within seconds, which is the *defence* side of first-seen (the first tx to reach majority stays in).
3. **Subscribe to DSProofs for the inputs you spend.** If the bot's outgoing tx gets double-spent by an attacker, you want to know within seconds so you can alert your operator or freeze a downstream action. The bot does not currently subscribe — **gap** — but should, for any tx above the dust threshold.

## 6. The "unconfirmed parent" gotcha

This is a **BCH-specific foot-gun** that the bot hit during the 2026-09-17 round-trip test documented in the zero-conf doc.[^bch-zero-conf]

### 6.1 The problem

You receive an unconfirmed UTXO (txid `X`) in your mempool. You immediately construct a new tx `Y` that spends the output of `X`. The node you broadcast `Y` to sees `X` in *its* mempool, validates `Y`, and accepts it. But another node — perhaps the one that mined `X`'s parent — may not have `X` in its mempool yet, because `X` was low-fee and evicted. When `Y` arrives, that node sees the input as *missing* and rejects `Y` with `"missing-inputs"` or `"bad-txns-in-belowoutpoints"`.[^child-relay]

### 6.2 Why this happens on BCH specifically

On BTC with package relay (PR #22674 and successors), a child can be submitted with its parents as a "package" and accepted atomically. **BCHN has no package relay.** Each tx is validated against the node's *current* mempool state, not against a speculative in-flight parent.

The result: a child tx may be accepted by *some* nodes and rejected by *others*, leading to propagation gaps. The child appears in block explorers' mempool views inconsistently, and the bot may see "broadcast succeeded" but the tx never actually propagates to miners.

### 6.3 What the bot does

The bot's `scripts/round-trip.mjs` test was constructed *specifically* to surface this gotcha. The preflight check:

```javascript
// scripts/round-trip.mjs ~line 30 (paraphrased from doc reference)
if (unconfirmed && !BCH_ALLOW_UNCONFIRMED_PARENT) {
  throw new Error('refusing to spend unconfirmed parent; wait for confirmation');
}
```

For the 1000-sat round-trip smoke test, the bot intentionally allowed the unconfirmed-parent spend because: (i) the sender was itself, (ii) the value was below the dust threshold of any rational attack, (iii) DSProofs would catch any conflict. **For larger or external unconfirmed-parent spends, the operational rule is: wait for at least 1 confirmation.** Spending an unconfirmed parent is a coin-selection smell, not a feature.

### 6.4 CPFP can rescue this

If a stuck parent is preventing a child from propagating, the child can be rebuilt with a higher fee — but the child must also be broadcast to nodes that have the parent in mempool. **The reliable workaround is to wait for the parent to confirm**, not to keep re-broadcasting the child.

## 7. CashTokens mempool considerations

### 7.1 What extra policy applies

CashTokens (activated May 2023, CHIP-2022-02-CashTokens) added two consensus rules that affect mempool validation:

1. **Token prefix parsing.** Every output now has an optional token prefix (`category: 32 bytes + amount: varint + optional NFT`). Parsing is mandatory; malformed prefixes are consensus-invalid (and thus rejected at mempool too).[^ct-spec]
2. **NFT commitment / capability rules.** An output can carry at most one NFT. The capability (`none`/`mutable`/`minting`) and commitment (≤40 bytes) are part of the token prefix.

Mempool policy does **not** add *new* dust, fee, or size rules beyond what BCHN already had. The dust threshold for a token output is computed dynamically (see §1.3) based on its actual byte cost, and a token output below that threshold is rejected.

### 7.2 What the bot does about tokens

`lib/tokens.mjs` handles three token-specific concerns:

1. **UTXO → token-prefix mapping** (`utxoToTokenPrefix`) — converts a Rostrum Electrum UTXO (with `token_data` field) to the libauth token-prefix shape used in `unlockingBytecode.token`. **Critical:** when signing, the token prefix from the *input* UTXO must be passed into the signing serialization preimage. Without it, the network rejects with `mandatory-script-verify-flag-failed` because the preimage the signer saw differs from the preimage the network verifies against.[^bot-sign]
2. **Token-aware UTXO selection** (`selectInputsForTokenSend`) — covered in §2.3.
3. **Dust-floor bumping** (`createTokenOutput`, `createNftOutput`) — if the requested sat amount is below `getDustThreshold`, the output is bumped to the dust floor.

### 7.3 Gas-vs-fee (no such concept on BCH)

There is no separate "gas" or execution cost for CashTokens operations. A CashScript contract spends the same relay + inclusion fee as any other tx; the script execution cost is borne by miners at block-validation time and does not affect fee policy. This is a meaningful difference from EVM chains (ETH, etc.) where complex operations are *more expensive*; on BCH, the cost of a tx is dominated by its byte size, not its computational complexity.

## 8. Operational rules the bot should enforce (summary)

| Situation | Rule |
|---|---|
| Constructing a P2PKH spend | Largest-first BCH-only UTXOs; 1000-sat safety margin over target[^bot-send] |
| Constructing a token send | Two-pass: FT-largest for token coverage; BCH-largest (BCH-only preferred) for sat coverage[^bot-tokens] |
| Computing fee | 1 sat/byte default; ~268 sats for typical 1-input/2-output P2PKH tx[^minrelay-ref] |
| Change < 546 sats | Absorb into fee; do not create a sub-dust change output[^bot-send] |
| Stuck tx | CPFP (spend a child output with elevated fee); do NOT broadcast an alternative tx (first-seen will reject)[^cpfp-ref] |
| Receiving unconfirmed UTXO from external sender | Wait 1 confirmation before spending; unconfirmed-parent spends are unreliable on BCHN due to lack of package relay[^child-relay] |
| Receiving 0-conf payment (small-ticket, in-person) | Accept with DSProof monitor; do not accept unattended deposits >$100 at 0-conf[^bch-zero-conf] |
| Output carrying tokens | Use libauth `getDustThreshold`; default `satsAmount = 1000n`; pass token prefix to signing serialization[^bot-sign] |
| Doubling up on a stuck tx | Do NOT; first-seen rule will reject the alternative. RBF does not exist on BCH.[^bch-zero-conf] |

## 9. Gaps and known unknowns

- **Exact BCHN `MAX_STANDARD_TX_SIZE` value in v29.1.0.** Standard inheritance is 100,000 vbytes; not re-verified against current source for this doc.
- **Bot does not subscribe to DSProofs** for its outgoing inputs. Should, for sends > dust threshold.
- **Bot does not implement CPFP** acceleration path.
- **Bot does not have a fee-estimation API client.** Hard-coded 1 sat/byte constant; no `estimatefee` query.
- **No mainnet stress data** for ABLA-era mempool behaviour (no large sustained fee spike since ABLA activated May 2024).
- **Libauth `getDustThreshold` exact formula** for CashTokens outputs — I cite the function but did not enumerate the bytes-bytes calculation here. The library is the source of truth.

## Sources

[^bot-network]: bch-bot `lib/network.mjs` (failover list of public Rostrum/Electrum servers, verified 2026-09-17 against Selene Wallet `src/util/network.ts`). <https://gitlab.com/selene.cash/selene-wallet>
[^bot-send]: bch-bot `scripts/send.mjs` (largest-first selection, 1000-sat safety margin, dust-absorption of change). Repo: `/home/luke/bch-bot/`.
[^bot-sweep]: bch-bot `scripts/sweep.mjs` (dust consolidation; SWEEP_THRESHOLD_SATS = 5000, MIN_REMAINING_AFTER_FEE_SATS = 500).
[^bot-tokens]: bch-bot `lib/tokens.mjs` (token-aware selection, dust bumping, FT-largest selection). DUST_RELAY_FEE = 1000n; libauth `getDustThreshold` integration.
[^bot-sign]: bch-bot `lib/sign.mjs` (signing serialisation: `tokenPrefix` must be passed into `unlockingBytecode.token` for token-bearing inputs).
[^minrelay-ref]: Bitcoin Optech "Default minimum transaction relay feerates": <https://bitcoinops.org/en/topics/default-minimum-transaction-relay-feerates/>; r/btc "How and when will BCH lower the minimum transaction fee?" <https://www.reddit.com/r/btc/comments/llc2fa/how_and_when_will_bch_lower_the_minimum/> (BCHN, BU, EC all default 1.0 sat/byte; Bitcoin Core PR #33106 later lowered BTC to 0.1 sat/byte but BCHN has not).
[^bnb-ref]: Bitcoin Core PR Review Club on BnB (Murch's algorithm): <https://btctranscripts.com/edgedevplusplus/2018/coin-selection>; Stack Exchange "How does the Branch and Bound coin selection algorithm work?" <https://bitcoin.stackexchange.com/questions/119919/how-does-the-branch-and-bound-coin-selection-algorithm-work>
[^knapsack-ref]: Bitcoin Stack Exchange "What are the trade-offs between the different algorithms for deciding which UTXOs to spend?" <https://bitcoin.stackexchange.com/questions/32145/what-are-the-trade-offs-between-the-different-algorithms-for-deciding-which-utxo/32445>
[^srd-ref]: Bitcoin Core PR #17526 "Add Single Random Draw as an additional coin selection algorithm" <https://github.com/bitcoin/bitcoin/pull/17526>; PR Review Club <https://bitcoincore.reviews/17526>
[^waste-ref]: Bitcoin Core PR #22009 "Decide which coin selection solution to use based on waste metric" <https://github.com/bitcoin/bitcoin/pull/22009>; PR Review Club <https://bitcoincore.reviews/22009>
[^abla-ref]: BCHN v27.0.0 release notes (CHIP-2023-04 Adaptive Blocksize Limit Algorithm, ABLA cap = 2 GB; activated May 15, 2024). <https://docs.bitcoincashnode.org/doc/release-notes/release-notes-27.0.0/>; Bitcoin Cash Podcast FAQ "What is the maximum Bitcoin Cash blocksize?" <https://bitcoincashpodcast.com/faqs/BCH/what-is-the-maximum-bch-blocksize>
[^fss-ref]: Bitcoin Stack Exchange "Double Spending before confirmation" — explains the first-seen default and what "First Seen Safe" rule means: <https://bitcoin.stackexchange.com/questions/44198/double-spending-before-confirmation>
[^cpfp-ref]: Lightspark "Understanding Child-Pays-For-Parent (CPFP)" <https://www.lightspark.com/glossary/child-pays-for-parent-cpfp>; r/BitcoinBeginners "Cancel unconfirmed parent" <https://www.reddit.com/r/BitcoinBeginners/comments/17w8bac/cancel_unconfirmed_parent/>
[^mempool-size]: Bitcoin Optech "Default maximum mempool size"; btq-core PR #184 "drop default mempool from 2 GB to 300 MB" <https://github.com/btq-ag/btq-core/pull/184>; Spark "Bitcoin Mempool Policy Reference" <https://www.spark.money/tools/bitcoin-mempool-policy-reference>
[^child-relay]: Stack Exchange "Accidentally spent from unconfirmed transaction" <https://bitcoin.stackexchange.com/questions/69937/accidentrally-spent-from-unconfirmed-transaction>; Bitcoin Core PR Review Club on package relay (PR #22674) <https://bitcoincore.reviews/22674> (BTC-only, not in BCHN)
[^ct-spec]: CashTokens CHIP specification <https://cashtokens.org/docs/spec/chip/>; Bitcoin Cash Network-Level Validation Rules <https://reference.cash/protocol/blockchain/transaction-validation/network-level-validation-rules>
[^bch-zero-conf]: `references/bch-zero-conf-security.md` in this wiki (covers DSProofs, no-RBF, First Seen Safe, merchant risk).
[^1]: Bitcoin Stack Exchange "transaction fees — Since Bitcoin Core 0.14.0, how does a node with default settings compute the dust limit?" <https://bitcoin.stackexchange.com/questions/57416/since-bitcoin-core-0-14-0-how-does-a-node-with-default-settings-compute-the-dus>
[^2]: Bitcoin Cash Protocol reference, Network-Level Validation Rules: <https://reference.cash/protocol/blockchain/transaction-validation/network-level-validation-rules>
[^3]: Bitcoin Stack Exchange "What is the dust limit on Bitcoin Cash transactions?" <https://bitcoin.stackexchange.com/questions/71604/what-is-the-dust-limit-on-bitcoin-cash-transactions> — "bitcoin cash dust limit is exactly the same as Bitcoin which is 546 Satoshi."
[^max-tx-gap]: **Gap** — Bitcoin Core's `MAX_STANDARD_TX_SIZE = 100000` is the documented inheritance. Not independently verified against BCHN v29.1.0 source for this doc.
[^reddit-relay]: r/btc discussion of BCH relay fee policy <https://www.reddit.com/r/btc/comments/llc2fa/how_and_when_will_bch_lower_the_minimum/>

---

*Authored by security-research subagent, lens: UTXO/mempool/double-spend. Cross-references `references/bch-zero-conf-security.md` (already in this wiki) for the acceptor-side 0-conf / DSProof / FSS discussion. Word count: ~2,800.*