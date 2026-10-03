---
pageType: reference
id: reference.swaps.pool-position-derivation
description: A Cauldron pool position is a UTXO at an address derivable from public data, not a route handed out by a router — and why treating a router's parent transaction as authoritative turns a normal race into a permanent failure.
sourceUrl: https://github.com/cashonize/cashonize-wallet/blob/main/src/utils/defi/cauldronPools.ts
---

# Pool positions are derivable, not handed out

The single most useful thing learned in three days of failing to swap, and it
came from reading one wallet's source rather than from probing our own stack
fifty more times.

From `cashonize/src/utils/defi/cauldronPools.ts`:

> A Cauldron pool is a contract written in raw BCH Script, not CashScript, and
> the only variable part of it is the 20-byte public key hash of the pool owner.
> All pools of one owner therefore sit at the same p2sh32 address, so listing an
> owner's pools is a **UTXO lookup on that address**.

## The contract is a constant

Raw BCH Script, not CashScript. The only variable is the owner's pubkey hash:

```
746376a914 <20-byte owner pkh> 88ac67c0d1c0ce88c25288c0cdc0c788c0c6c0d095c0c6c0cc9490539502e80396c0cc7c94c0d3957ca268
└─ OP_DEPTH OP_IF OP_DUP OP_HASH160 PUSH20
                                                            └─ OP_EQUALVERIFY OP_CHECKSIG OP_ELSE <CPMM conditions> OP_ENDIF
```

Wrapped in p2sh32 by hashing the whole script. So a pool's on-chain home is a
function of public data:

```js
const contract = hexToBin('746376a914' + ownerPkh + '88ac67c0d1c0ce88c25288c0cdc0c788c0c6c0d095c0c6c0cc9490539502e80396c0cc7c94c0d3957ca268');
const locking  = encodeLockingBytecodeP2sh32(hash256(contract));
```

**There is no artifact and no CashScript instance to fetch.** A hand-rolled
lookup needs nothing but the template and a hashing function.

Cashonize asks its own Electrum server rather than the Cauldron indexer, for a
reason worth copying:

> The lookups go to the wallet's own electrum server… The Cauldron indexer
> answers the same question from an owner public key hash, but that would hand
> the wallet's list of addresses to a third party.

## The position moves; the pool does not

This is the part that took three days to see. Every swap **spends the old
outpoint and re-creates the same covenant at a new one.**

So a route built thirty seconds ago names coins that no longer exist, while the
liquidity sits there completely untouched. The covenant is the durable identity;
the outpoint is a per-swap detail.

Measured on mainnet against the parent the router kept serving us,
`fd02de7d…1138`, whose 56 outputs are all consumed:

```
distinct pool covenants in the drained parent: 39
covenants that STILL hold a live position:     38/39

  lock aa20d59cda2cd6547b5d..   3 live   freshest h971308
  lock aa208bb9d9cf03ca1a61..  60 live   freshest h971321
  lock aa2053da08483a5df8ed..   1 live   freshest h971321
```

Those heights are within a minute of the chain tip. **The pools were never
dead.** Luke said swaps were happening on Cauldron while I had concluded the
protocol was broken; he was right, and this is the measurement that proves it.

## The bug this exposes: a parent is not a position

Our swap path takes the router's parent transaction as authoritative and spends
named outpoints of it. The chain does not agree that a parent is what a pool is.
A pool is a covenant address holding whatever position is current.

The consequence is a category error in how failures are read:

| Question | Answer via outpoint | Answer via covenant |
|---|---|---|
| Is the liquidity real? | no — outpoint gone | **yes — 38/39 covenants live** |
| Is the route still valid? | no | **no, but the pool is** |
| Correct response | refuse | re-derive, then race |

**Refusing is the wrong response to a position that has moved rather than
vanished.** A swap through a busy pool is an ordinary race — someone else can
consume the position between your build and your broadcast, and a retry can win
that. Pinning to a parent transaction converts a winnable race into a
permanent, 100%-reproducible failure, because a fresh quote names the same
drained parent and a live one names a parent that is drained by the time it
arrives.

The 1ms measurement settles that this is not merely a timing problem:

```
[probe] build returned 1791045048957
[probe] check start   1791045048958      ← one millisecond later
```

One millisecond. There is no window to close and no contention to out-wait. The
positions were already gone when the router handed them over.

## The fix, in one line of intent

For each pool covenant the router names, ask the token-aware node **where that
position lives now** and use the live outpoint.

The owner's pubkey is not even required for this: a p2sh32 locking script
already contains the hash of its redeem script, so the covenant's identity is in
the output the router gave us. One `listunspent` per distinct lock is enough.

That turns a permanent failure into the race it was always going to be, and it is
a change to the swap path rather than to the gate — the gate's exact-outpoint
check is correct and should stay.

## Related

- [`../entities/index.md`](../../entities/index.md) — the DEX itself, and the
  constant-product mechanics
- [`cauldron-k-invariant.md`](cauldron-k-invariant.md) — the two rules a valid
  route must satisfy, and why the published formula does not reproduce
- [`gates-that-pass-when-they-should-fail.md`](gates-that-pass-when-they-should-fail.md)
  — the gate that reported `12/12 verified` on a drained route
- [`../security/dex-swap-integration.md`](../../security/dex-swap-integration.md) —
  what an integrating wallet must verify before signing
