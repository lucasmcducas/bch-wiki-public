# Swaps on BCH

Everything about trading on a CashTokens AMM: how a route is built, what a pool
actually is on-chain, what a wallet must verify before it signs, and the failure
modes that only show up against a live pool.

This is a cross-cutting section. The pages that matter most were written after a
three-day investigation in which a swap could be built and signed correctly and
still be rejected by the network — and in which the first five explanations for
why were all wrong.

## Read in this order

1. **[Pool positions are derivable, not handed out](pool-position-derivation.md)**
   — the finding that reframed everything. A pool is a UTXO at an address you
   can compute from public data; a router naming a parent transaction is not the
   same thing as naming a live position.
2. **[The Cauldron k-invariant and the re-creation rule](cauldron-k-invariant.md)**
   — the two rules the specification states, the doc-vs-implementation
   discrepancy in the k formula, and the byte-order trap.
3. **[Gates that pass when they should fail](gates-that-pass-when-they-should-fail.md)**
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

## Trading today: `bch-bot swap-open`

The in-wallet swap path builds a correct transaction and cannot get it accepted,
for the reason on the first page. So `bch-bot swap-open <sell> <buy>` opens the
Cauldron app instead, which is what the reference wallet does:

```
bch-bot swap-open BCH pusd
  opened https://app.cauldron.quest/swap/2469acc5afa4b10c...
  selling    BCH
  receiving  PUSD (2469acc5afa4b10c..)
```

Two limits, both in `--help` rather than discovered later:

- **BCH cannot be a destination.** The app's URL is keyed on a CashToken
  category, and BCH is the native asset with no category, so there is no page to
  open. Selling a token for BCH means using the app directly. Cauldron trades the
  pair both ways, so this is a limit of the URL scheme, not the protocol.
- **The amount is typed in the browser.** The app reads no query parameters
  beyond the category in the path.

The browser signs. The command never touches a private key and never moves
funds — which is the trade being made: the wallet stops being the signer for
swaps specifically, in exchange for a path that works.

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

<!-- openclaw:wiki:swaps:index:end -->
