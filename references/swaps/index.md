# Swaps on BCH

Everything about trading on a CashTokens AMM: how a route is built, what a pool
actually is on-chain, what a wallet must verify before it signs, and the failure
modes that only show up against a live pool.

This is a cross-cutting section. The pages that matter most were written after a
three-day investigation in which a swap could be built and signed correctly and
still be rejected by the network — and in which the first five explanations for
why were all wrong.

## Read in this order

1. **[In-wallet swaps: ExchangeLab and the public indexer](../in-wallet-swaps.md)**
   — what actually works now: pool state from a public indexer, the transaction
   assembled locally, no server in the middle.
2. **[Pool positions are derivable, not handed out](pool-position-derivation.md)**
   — the finding that reframed everything. A pool is a UTXO at an address you
   can compute from public data; a router naming a parent transaction is not the
   same thing as naming a live position.
3. **[The Cauldron k-invariant and the re-creation rule](cauldron-k-invariant.md)**
   — the two rules the specification states, the doc-vs-implementation
   discrepancy in the k formula, and the byte-order trap.
4. **[Gates that pass when they should fail](gates-that-pass-when-they-should-fail.md)**
   — a safety check that printed `12/12 verified unspent` on a route whose every
   input was already consumed, and the three questions that catch this class.

## Also relevant, elsewhere in the wiki

- [`../security/dex-swap-integration.md`](../../security/dex-swap-integration.md) —
  what an integrating wallet must verify locally before signing: the
  operator-signing problem, the router's 10 bps fee output, the assembler safety
  gate, and CPMM slippage semantics
- [`../security/pre-signing-invariant.md`](../../security/pre-signing-invariant.md) —
  the pre-signing checks, and why "signed" and "safe to sign" are different claims
- [`../entities/`](../../entities/index.md) — the DEX, libauth, PUSD and the other projects
- [`../syntheses/failure-modes-we-hit.md`](../../syntheses/failure-modes-we-hit.md)
  — the running ledger, including five retracted explanations for one rejection
- [`../sources/cauldron-docs-swap.md`](../../sources/cauldron-docs-swap.md) —
  excerpts from Riften's specification, the primary source for every rule stated here

## Why this is its own section

Swaps are where a BCH wallet meets every hard part of the stack at once: p2sh32
covenants that ordinary Electrum nodes do not index, a constant-product AMM
whose invariant has to hold across a dozen chained pools, an external router
whose word is not the chain's, and a mempool where someone else can take your
inputs between your build and your broadcast.

Every one of those produced a false reading during the investigation. A check
that could not see covenants reported live positions as spent. A client library
turned a real rejection into an empty object. A gate scoped to a container
instead of a position reported success on a dead route. The pages here are
mostly about that class of problem, because it is the part that is genuinely
hard to get right and the part that generalises past swaps.

## Signing in the app: WizardConnect

The browser is a UI, not the signer. `app.cauldron.quest` speaks
[WizardConnect](../wizardconnect/index.md), which lets a wallet on another
machine sign the transaction the app builds — our key, our verification
gates, the working router's route.

## Trading today: `bch-bot swap`

The in-wallet path works. The transaction is assembled locally by
`@cashlab/cauldron` from pool state read off `indexer.riften.net`, the wallet
signs its own input, and the wallet broadcasts:

```
bch-bot swap BCH pusd 0.001
  [1/4] quote: 0.31 PUSD across 2 pool(s)
  [2/4] 3 UTXO(s) available: 3 coin (1656311 sat)
  [3/4] built 636 bytes, 2 payout(s), fee 636 sats (already signed)
        verified: pays 0.31 PUSD against a quote of 0.31
  DRY RUN -- set BCH_CONFIRM=yes to broadcast
```

`swap-open` is retired. Opening the browser was a workaround for a router that
named a spent parent; with the build local there is nothing to work around.

**The router was never the bug — a server-side build was.** Re-pointing its
inputs would have meant reimplementing the constant-product math, which is what
`@cashlab/cauldron` turns out to be. See
[../in-wallet-swaps.md](../in-wallet-swaps.md).

## The uncomfortable summary

The investigation produced six explanations for one rejection. Five were wrong:

| Claim | What killed it |
|---|---|
| Pool contention | The check ran 1 ms after the build — no window exists |
| Nonexistent prevouts | The same queries denied known-confirmed transactions |
| Unsigned transaction | Only our 2 inputs are signed, by design |
| Invalid output index | The re-creation rule holds: `in[i] → out[i]` |
| The node cannot do covenants | The raw frame said `Missing inputs`, not a script error |
| *The router serves a drained parent* | **correct — and the pools were live all along** |

The last one survived only because someone asked how other wallets do it. Every
earlier step had been spent interrogating our own stack.

And the answer to that question changed the architecture rather than just
explaining the failure. Paytaca builds its own swaps locally with the same
published SDK, from the same public indexer this wallet already used for token
lookup. Two days of work went into re-deriving something that was a dependency
all along — which is the argument for reading a reference implementation before
reimplementing a protocol.

<!-- openclaw:wiki:swaps:index:end -->
