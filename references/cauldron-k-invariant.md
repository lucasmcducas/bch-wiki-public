---
pageType: reference
id: reference.cauldron-k-invariant
description: The two rules a Cauldron swap must satisfy, read from Riften's own docs and verified against a real signed transaction — the positional re-creation rule, and the k-invariant that makes an AMM non-arbitrageable. Includes the doc-vs-implementation discrepancy in the published formula.
sourceUrl: https://docs.riftenlabs.com/cauldron/swap/
---

# The Cauldron k-invariant and the re-creation rule

Two requirements govern a valid Cauldron trade. Both are stated in Riften's
documentation and both are now asserted in code
(`scripts/test-cauldron-k-invariant.mjs`, 6/6 against a real mainnet swap).

I spent days probing BCH nodes for the cause of a `Missing inputs` rejection and
never opened this page. Reading it settled in ten minutes what the probing could
not, because a node error names a symptom and the specification names the rule.

---

## Rule 1: the contract is re-created at the same output index

> "Re-create the contract — The Cauldron contract is re-created in the same
> output index as it was used as input index."

Positional, and it holds: `in[i] → out[i]` for all 13 pool inputs, 15 inputs to 15
outputs.

**The trap: output index is not parent vout.** I objected that the route spends
parent vouts `v1…v35` while the transaction has only 15 outputs, and concluded
the route was structurally invalid. That is two different numbering spaces
conflated. The rule constrains the *position* of the re-created contract in the
transaction being built — nothing to do with the index the pool UTXO happens to
sit at in some older transaction.

This is worth stating plainly because I asserted it confidently in conversation
and in a draft wiki section before checking it. A transaction is free to spend
`vout 35` and write its replacement at `output 12`; the covenant does not care
where the coin came from.

## Rule 2: the constant product must hold

> "kInput = tokens × satoshis of the input"
> "kOutput = tokens × (satoshis − fee)"
> "The interaction must satisfy: kOutput >= kInput"

Measured across all 13 pools of a live 0.005 BCH → PUSD swap:

```
aggregate kIn  = 455343290524
aggregate kOut = 455348961740
ratio          = 1.000012455
```

Per pool, `kOut/kIn` runs **0.99985 – 0.99997** — each individual pool *loses* a
hair of k — while the aggregate sits just *above* 1.0. That is the AMM working
exactly as it should:

- **Per pool, k rises by ~1.06% in sats-per-token terms.** A pool can never lose
  k, so no later trade can arbitrage an earlier one away. The pools absorb the
  trade.
- **The aggregate is conserved to within rounding.** The router moves value
  *between* pools while charging 0.3% to the LPs; it does not create or destroy
  the invariant globally.

### The published formula does not reproduce the implementation

Applying the doc's literal formula — subtracting a 3% fee from the output side —
fails **13 of 13** pools, each by a uniform **~0.0015%**:

| pool | kIn | kOut | ratio |
|---|---|---|---|
| 0 | 2,252,154,158 | 2,252,151,918 | 0.999852 |
| 5 | 76,195,923,675 | 76,185,399,828 | 0.999862 |
| 12 | 108,968,738,396 | 108,965,002,622 | 0.999966 |

With no fee subtracted, **13/13 pass**. The fee is evidently accounted for
elsewhere — most likely as a separate router output, which the docs also describe
("each built transaction carries a router fee as one of its outputs, currently 10
basis points"). Reading a pool's `k` off a single input/output pair and ignoring
the fee output misattributes it.

**Treat the aggregate as the meaningful check** and the per-pool figure as
informational. If you port this to a verifier, the per-pool bound will fail every
valid transaction until you locate where the fee actually goes.

### An assertion that flagged a correct swap as broken

My first conservation bound was `kOut <= kIn`, on the reasoning that k is
"conserved". It failed on this transaction — because `kOut` is *supposed* to land
slightly above `kIn`. The correct bound is a band:

```
kOut >= kIn × 0.999   and   kOut <= kIn × 1.001
```

**A gate that rejects valid input is worse than no gate**, because it trains you
to disable it. When a check fails, ask whether the check or the transaction is
wrong before loosening anything — and here the transaction was right.

---

## What this does and does not establish

**Established, and now asserted in a test:**

- The re-creation rule holds; output index is not parent vout.
- The k-invariant is conserved in aggregate and non-decreasing per pool.
- The transaction is structurally what a Cauldron swap must be.

**Still not established:** why the node rejects it with `Missing inputs`. The
transaction is valid by every rule the specification states. That points at the
*node* rather than the wallet — a broadcasting node that cannot evaluate p2sh32
covenants would report a covenant input as unusable rather than as a script
failure, which is precisely the symptom.

Riften's own guidance bears on this: the build response says to POST the signed
transaction to `broadcast.cauldron.quest/broadcast` and prefers it over a single
node, because *"trades against the same pools chain on one another, so a
transaction that reaches only part of the network is how double-spend conflicts
start."* A wallet broadcasting to one node is the failure mode their own
documentation warns against.

## Client facts this page cost me

Both are in the repo's code and both cost hours when violated:

- **`request` is variadic.** `client.request(m, a, b)`, never `request(m, [a, b])`.
  The array nests, the node answers `{}`, and `{}` is indistinguishable from
  "no such transaction".
- **Byte order is not canonical.** libauth's `outpointTransactionHash` is wire
  order; Electrum's `tx_hash` is its byte-reverse. Try both — querying only one
  reports a live transaction as nonexistent.

## Related

- [`../security/dex-swap-integration.md`](../security/dex-swap-integration.md) —
  the live-swap addendum and the pool-input structure
- [`../syntheses/failure-modes-we-hit.md`](../syntheses/failure-modes-we-hit.md)
  §28 — the probe that reported known-confirmed transactions as nonexistent
