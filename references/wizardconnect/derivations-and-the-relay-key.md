---
pageType: reference
id: reference.wizardconnect.derivations
description: The four HD chains a WizardConnect wallet derives, why the relay secret gets a chain of its own, and the assertions that keep it from ever reaching a dapp.
sourceUrl: https://gitlab.com/riftenlabs/lib/wizardconnect
---

# Derivations and the relay key

A wallet adapter is almost entirely derivation. A wrong derivation is a wallet
that signs for the wrong chain — which either fails loudly, or worse, quietly
shares keys with a dapp. So this page is mostly about the one derivation that is
genuinely subtle, and about how to *assert* the properties rather than trust
them.

## Four chains, domain separated

Under the wallet's BIP44 parent, `m/44'/145'/0'`:

| Child | Chain | Exposed to the dapp? |
|---|---|---|
| `0` | Receive | yes, as an xpub |
| `1` | Change | yes, as an xpub |
| `7` | Cauldron — the dapp's defi chain | yes, as an xpub |
| `8` | **relay secret, child 0** | **never** |

`7` is the interesting one for swaps: a dapp spends coins on its own chain, so a
Cauldron route's inputs live somewhere the wallet would never otherwise scan.
The reference wallets expose it as a fourth named chain for exactly that reason.

## The relay secret, and why it is the security-critical one

The spec's reasoning, verbatim from the reference implementation:

> Relay identity keys are HMAC(dedicated chain key, pairing URI): deterministic
> per URI so reconnects restore the same Nostr identity, distinct per URI so
> dapps can't correlate sessions. **Chain index 8 keeps the relay secret off the
> xpub chains shared with dapps.**

```js
const relaySecretNode = deriveHdPrivateNodeChild(deriveHdPath(hdNode, `m/44'/145'/0'/8`), 0);
getRelayPrivateKey: (uri) => hmacSha256(nodes.relayHmacKey, utf8ToBin(uri)),
```

Two requirements pull against each other, and both are met:

**Deterministic per URI.** A reload must not orphan a session, so the same
`wiz://…` URI always yields the same Nostr identity. If it were random, every
page refresh would look like a new wallet and a dapp would lose its pairing.

**Unlinkable across URIs.** A dapp must not be able to tell that two sessions
belong to the same person. If the key were derived from a fixed value, every
session a user ever had would share one public key and would be trivially
correlatable — by us, and by the relay.

HMAC under a per-URI domain gives both. And **not being derivable from any xpub
the dapp holds** is what makes the unlinkability real rather than a promise:
the dapp has nothing to correlate *against*, because the pubkey's parent is on a
chain it never receives.

## Assertions, not comments

The properties above are testable, so test them. This matters more than usual
because the first version of this test suite **passed while proving nothing**:

```js
// USELESS: a private key and an xpub can never be equal, so this is always true
check('relay key is not any spend-chain key', !xpubs.has(adapter.getXpub(...)));
```

A test that cannot fail is worse than no test, because it reads as coverage. The
real assertions:

```js
check('the relay chain is NOT one of 0/1/7',        ![0,1,7].includes(RELAY_CHAIN));
check('getXpub on the relay chain throws',          throws(() => adapter.getXpub(RELAY_CHAIN)));
check('getPublicKey on the relay chain throws',     throws(() => adapter.getPublicKey(RELAY_CHAIN, 0n)));
check('deterministic: same URI, same key',          hex(kA) === hex(kAagain));
check('distinct per URI: sessions uncorrelatable',  hex(kA) !== hex(kB));
check('differs from a spend key on a shared chain', hex(kA) !== hex(spendKey0));
```

`getXpub(8)` throwing is the load-bearing one. It is a **structural** guarantee
rather than a review promise: there is no code path by which the relay chain can
leave the wallet, because the lookup refuses the index.

The same reasoning applies to a hardened index. A dapp derives addresses from an
xpub, and an xpub **cannot** do hardened derivation, so a request naming a
hardened index is malformed by construction:

```js
check('rejects a hardened index', throws(() => adapter.getPublicKey(0, 0x80000000n)));
check('rejects a negative index',  throws(() => adapter.getPublicKey(0, -1n)));
```

Rejecting it is not pedantry. Silently truncating a `bigint` to a `Number` to
make it fit would derive a *different, real* address.

## Domain separation is the whole discipline

Every value in a wallet that can be derived should be derived from exactly one
place, and that place should be named. The relay secret, the spend keys, the
xpub chains — four uses, four different derivations, zero overlap. The bug this
prevents is not exotic: it is a `childIndex` that silently defaults to 0 and
makes the relay secret the first receive key.

## What a wallet must never do

- Derive the relay identity from a spend path
- Persist it — it is recomputed per process from the seed, never written
- Expose chain 8 through `getXpub` or `getPublicKey`
- Accept a hardened or out-of-range index by coercion

## Related

- [The signing rules](signing-rules.md) — what happens after derivation
- [What WizardConnect is](what-it-is.md)
