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

## The actual cause: the pool UTXOs are already spent

Reading the raw JSON-RPC frame instead of the client library's return value
gives the node's actual verdict:

```
broadcast <our tx>  ERROR code=-32000
  "RPC error (-32602 InvalidParams): rejected by network; RPC error
   (-32000 Other): Call 'sendrawtransaction' to full node failed: Missing inputs"

broadcast "00"      ERROR -32603 "failed to parse tx"
```

Two different failures. **`Missing inputs` means the outpoint is absent from the
UTXO set** — not a covenant-evaluation error, and not a parse error. The node
parsed the transaction fine and passed it to the full node, which could not find
the coins.

The parent transactions all exist, so they are not unknown transactions. Their
outputs are consumed:

```
in[ 0] v 4   spent / absent
in[ 1] v 5   spent / absent
...
in[12] v 0   spent / absent
in[13] v 0   UNSPENT (1000 sats, h968967)     <- ours
in[14] v 0   UNSPENT (800000 sats, h971038)  <- ours
```

`h968967` matches what the wallet's own `utxos.mjs` reports — that agreement is
the control that makes this reading trustworthy.

**It is not a stale quote.** A route built seconds later (0.002 BCH, 12 pools)
spends the same dead parent. The router is building through pools whose outputs
are already gone, and `route.quote` accepts only `sell`/`buy`/`amount`/`side` —
there is no pool-selection parameter, so a wallet cannot steer around it.

### The client bug that made this invisible

`blockchain.transaction.broadcast` appears to return `{}` for everything. It
does not: **the client library collapses the JSON-RPC error object into an empty
object.** The node was answering the whole time, and the error text was lost in
the transport.

I compounded it for an hour by treating `{}` from a garbage one-byte hex as proof
the node was not evaluating anything. The two failures return the *same* `{}` at
the library boundary while being completely different errors underneath.

**Send the frame by hand when the answer matters.** `@electrum-cash/network`
swallows the error; a raw `ws.send(JSON.stringify({jsonrpc, id, method,
params}))` shows the code and message.

### `outpoint_hash` is not `tx_hash`

A `listunspent` entry carries both, and they differ — `outpoint_hash` is the
byte-reversed form:

```
pos=0 val=1000   outpoint_hash=3d9592aeafe46bb8cc   tx_hash=ab90acba4e383b3cc4
```

Matching on the wrong field reports every coin as spent. I hit this and briefly
concluded the wallet's own inputs were gone. Always match against **both**
orderings, and confirm with a known-good `utxos.mjs` reading before believing a
"spent" verdict.

Pinned by `scripts/test-swap-prevout-liveness.mjs` (4/4).

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
