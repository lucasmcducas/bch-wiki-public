---
pageType: entity
entityType: service
id: entity.riften-router
description: The Cauldron Router — Riften Labs' public no-auth WebSocket service that quotes trades, assembles UNSIGNED swap transactions (including the pool inputs no user can sign), and returns the prevouts needed to sign. This is what makes a self-custodial BCH swap executable at all.
sourceUrl: https://docs.riftenlabs.com/router/
---

# Riften Labs Cauldron Router

> The service that solves the central problem with swapping on Cauldron: a pool
> input must be signed by the **pool operator's** key, which no user wallet
> holds. The Router assembles the transaction, the wallet signs only its own
> inputs, and the wallet broadcasts.

## Why it exists

A Cauldron swap spends a pool's contract UTXO. That input is committed to the
pool operator's public key hash, embedded in the pool's locking bytecode
(per [Cauldron DEX](cauldron-dex.md)). So a plain wallet **cannot** produce a
valid swap transaction on its own — not with a better key manager, not with
more compute. The signing key simply is not the user's.

This is why `bch-bot` originally could not swap: its first attempt used a
placeholder key (`new Uint8Array(32)`) where the operator key belonged, which
would have produced a structurally valid but cryptographically meaningless
transaction.

The Router is Riften Labs performing that co-signing as a service. Riften Labs
operates the Cauldron DEX (see [Cauldron DEX](cauldron-dex.md) — the same
organisation runs Delphi the oracle and Moria), so they are the party whose
keys are needed anyway.

## Endpoint and methods

```
wss://router.riften.net/v1/route
```

Electrum-style JSON-RPC over a single WebSocket. No credentials during the
beta. Rate limited — a request over the limit is rejected with `rate_limited`
before any routing work happens.

| Method | Purpose |
|---|---|
| `route.quote` | one-shot price for a trade |
| `route.subscribe` | live quote with push updates (`route.unsubscribe`) |
| `tx.build` | assemble an **unsigned** swap transaction |
| `tx.subscribe` | live unsigned tx, re-pushed as pools move (`tx.unsubscribe`) |

Amounts are **integer strings** in base units (JS-safe). Prices are decimal
strings. Token categories and txids are 32-byte hex in display order. An asset
is either the literal `"bch"` or a 64-char hex category.

### Quote

```json
{"id":1,"method":"route.quote","params":{
  "sell":"bch",
  "buy":"2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544",
  "amount":"10000","side":"sell"}}
```

`side` says which number is fixed — `"sell"` spends exactly `amount`,
`"buy"` receives exactly `amount`. The two produce different transactions for
the same pair of numbers, so a caller must say which it means.

Response includes `input_amount`, `output_amount`, `market_pre_price`,
`market_post_price` (price impact), and `pools` (how many pool UTXOs were
consumed). Verified live: 0.01 BCH → PUSD routed across 28 pools.

### Build

Required: `sell`, `buy`, `amount`, `funding` (1–500 UTXOs), `receive_addr`,
`change_addr`. Optional: `min_output` (slippage floor, **sell mode only** — in
buy mode the output is what you asked for, so there is nothing to protect).

Response fields that matter:

| Field | Why it matters |
|---|---|
| `unsigned_tx_hex` | the assembled transaction; sign and broadcast it |
| `inputs_to_sign` | **the input indices the client owns** — sign only these |
| `source_outputs` | prevouts index-aligned to *all* inputs |
| `expected_output` | the slippage reference to check the build against |
| `fee_sats` / `fee_token_amount` | router fee, `"0"` where not applicable |
| `miner_fee_sats` | network fee estimate |

`source_outputs` is not optional in practice — see
[BCH signing and verification](bch-signing-and-verification.md).

## Cost

**The Router charges a fee: currently 10 basis points (0.1%) of the trade**,
paid as its own on-chain output. That is *in addition to* Cauldron's 0.3% LP
fee, so a swap costs roughly **0.4% total**.

The docs are explicit that the rate may change during the beta and that
callers must read `fee_sats` / `fee_token_amount` from each build rather than
hard-coding 0.1%, and that removing the fee output is grounds for being banned
from the service.

Commercial use: `hello@riftenlabs.com` for higher limits or a dedicated
endpoint.

## Broadcasting

The Router **never broadcasts**. Its view of the pools only advances from what
it observes on the network, so sending the transaction is the client's step:

```
POST https://broadcast.cauldron.quest/broadcast
Content-Type: application/json
{"tx": "<signed transaction hex>"}
→ {"txid": "94a933a0..."}
```

Use this rather than a single node: trades against the same pools chain on one
another, so a transaction that reaches only part of the network is how
double-spend conflicts start.

## Safety gate the Router applies

Before returning a transaction, the assembler checks:

- no token burn — every category conserved (Σ in == Σ out)
- sane miner fee — between the relay floor and a max-overpay band
- fee output present when a fee was charged
- no unspendable outputs — non-empty scripts, above dust

A failure returns `build_failed` rather than an unsafe transaction.

Errors: `bad_request`, `route_failed`, `insufficient_funds`, `build_failed`,
`slippage`.

## Stability

**Beta.** Methods, message shapes, and availability may change without notice,
and Riften states that quotes should be verified against the returned
transaction before signing. That instruction is not a formality — it is why
`bch-bot` compares `expected_output` against the quote and refuses to sign on
mismatch.

## Other Riften Labs services worth knowing

| Service | Use |
|---|---|
| Cauldron Router | quotes + unsigned swap assembly (above) |
| Indexer — `https://indexer.riften.net` | token/pool metadata; see below |
| Broadcast API | submit a signed tx to the network |
| Delphi | price oracle |
| Moria | Liquity-style borrowing |
| WizardConnect | xpub-based wallet connection protocol |
| Cauldron mobile app | reference UX; built-in swaps |

### Indexer endpoints

The indexer host is `indexer.riften.net` and **token endpoints are namespaced
under `/cauldron/`** — the bare `/tokens/...` path returns 404.

```
GET /cauldron/tokens/search_cached?q=pusd
GET /cauldron/tokens/list_cached?limit=50&by=tvl&order=desc
GET /cauldron/tokens/list_cached_by_ids?ids=<hex,hex>
GET /valuelocked/<token>
GET /volume/<token>
```

`search_cached` returns an array; the useful fields are `token_id`,
`display_name`, `display_symbol`, and `bcmr.token.decimals`. Note this is a
different shape from the Electrum `blockchain.token.list` method, and the
Electrum token methods are **not served by every public Rostrum node** — a
mainnet public node returned `{}` for all of them. Use the indexer.

## What this replaces

The hand-rolled Cauldron integration in `bch-bot` (~277 lines of pool parsing
and CPMM quoting, with a placeholder operator key) was deleted in favour of a
client for this service. The DEX operator maintains the router; a client
re-implementing their pool discovery has no upside and a maintenance cost.
