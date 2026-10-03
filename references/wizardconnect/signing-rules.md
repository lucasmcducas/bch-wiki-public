---
pageType: reference
id: reference.wizardconnect.signing-rules
description: The fixed SIGHASH_ALL|UTXOS|FORKID sighash, the two placeholder conventions for CashScript pubkeys and signatures, and a P2PKH encoding bug that executes and therefore passes a superficial check.
sourceUrl: https://gitlab.com/riftenlabs/lib/wizardconnect
---

# The signing rules

Three rules, none of them negotiable per request, and one encoding subtlety that
cost an afternoon.

## The sighash is fixed by the protocol

```
SIGHASH_ALL | SIGHASH_UTXOS | SIGHASH_FORKID     (0x41)
Schnorr signatures
```

The reference implementation's reasoning, which is the argument for why it is
mandatory rather than a default:

> Because the dapp selects which keys sign which inputs (inputPaths), this
> hashType is what makes that safe: the signature commits to the full
> transaction and all source outputs, so it cannot be grafted onto a different
> transaction and the dapp cannot misrepresent input amounts.

Read that as two separate protections:

**`ALL`** commits to every output. The dapp cannot sign one transaction and have
it be interpreted as another with different recipients.

**`UTXOS`** commits to every input's value. A dapp cannot show you "you are
sending 0.001 BCH" and broadcast "you are spending 10 BCH". This is the one
that matters for a swap, where the output amount is the entire point.

And there is a free side effect worth knowing: because `UTXOS` is set, **a
wrong-key signature is simply invalid**. So a wallet does not need to look up
which key owns an input in order to refuse it —

> The derived key is not checked against the locking bytecode: with
> SIGHASH_UTXOS a wrong-key signature is simply invalid.

That saves a round trip and removes a place where a bug could hide.

## Placeholders: the dapp cannot know your pubkey

CashScript contracts take a pubkey and a signature as constructor and function
arguments. A dapp works in cashaddresses, which encode pubkey **hashes**. So it
cannot know either value, and it has to leave slots:

> We signal the use of pubkeys by using a 33-byte long zero-filled arrays and
> schnorr (the currently supported type) signatures by using a 65-byte long
> zero-filled arrays. Wallet detects these patterns and replaces them
> accordingly.

Two literal byte patterns, in the unlocking bytecode of a contract input:

| Pattern | Meaning | Replace with |
|---|---|---|
| `41` + 65 zero bytes | signature slot | `41` ‖ signature ‖ sighash byte |
| `21` + 33 zero bytes | pubkey slot | `21` ‖ compressed pubkey |

```js
const SIG_PLACEHOLDER    = `41${'00'.repeat(65)}`;
const PUBKEY_PLACEHOLDER = `21${'00'.repeat(33)}`;
```

**The sighash's `coveredBytecode` for a contract input is the redeem script**,
not the locking script. This is the detail that decides whether a covenant
signature is valid, and it is invisible in the placeholder pattern. For a
Cauldron p2sh32 pool that redeem script is the whole contract, and getting it
wrong produces a signature that is simply rejected.

The wallet must **throw** rather than continue in three cases:

- a placeholder is present but `inputPaths` named no key for that input
- a contract input has no `redeemScript`
- `inputs.length !== sourceOutputs.length`

That last one is worth keeping as an assertion rather than a comment. It is the
cheapest possible check that the two parallel arrays really are parallel, and
mismatched pairs produce a sighash over the wrong output — silent, and
indistinguishable from a bad signature.

## The P2PKH encoding trap

A real bug from building this, and the most interesting failure in the whole
project.

A P2PKH unlocking script is **two pushes**:

```
<signature + sighash>   <pubkey>
   0x41 + 65 bytes        0x21 + 33 bytes
```

The first version concatenated them into **one 68-byte push**:

```js
// WRONG
input.unlockingBytecode = hexToBin(`44${binToHex(signature)}${binToHex(pubkey)}`);
```

And here is why that is instructive: **it executes.** A single push of
`sig ‖ sighash ‖ pubkey` is a valid scriptSig — the sig lands where the stack
expects it, and the pubkey is right behind it. So a test that checked "does it
sign" and "is the sighash byte present" both passed.

It is still wrong. It is not the encoding the wallet uses, a node applying the
standard template is entitled to reject it, and it is the kind of thing that
works in testing and fails on mainnet. The wallet's own signer emits two pushes:

```js
scriptSig[at++] = pubkey.length;   // lib/sign.mjs
scriptSig.set(pubkey, at);
```

The lesson generalises past this protocol: **a test that only asks "does it
work" cannot catch a wrong encoding, because a wrong encoding often does work.**
Assert the structure, not the outcome:

```js
check('unlocking bytecode is push-65 (0x41) then push-33 (0x21)',
  /41[0-9a-f]{130}21[0-9a-f]{66}/.test(signedTransaction));
```

## Guards worth having even when they look redundant

```js
if (!Array.isArray(sourceOutputs))            throw new Error('sourceOutputs is required');
if (inputs.length !== sourceOutputs.length)   throw new Error('length mismatch');
if (!coveredBytecode)                         throw new Error('include contract redeemScript');
if (!signingKey)                              throw new Error(`no key for input ${i}`);
```

A wallet that signs something it cannot fully describe is not a wallet. Each of
these turns an unexplained bad signature into a named failure.

## What a signing UI has to show

Not code, but it is the part that makes the protocol safe in practice: which
dapp, which inputs are ours, and the **net effect in the token being spent**. A
covenant input whose decoded value is not BCH — a pool position, a receipt — has
to be labelled as such, or the dialog is asking the user to approve something
they cannot read.

## Related

- [Derivations and the relay key](derivations-and-the-relay-key.md)
- [`../swaps/cauldron-k-invariant.md`](../swaps/cauldron-k-invariant.md) — the
  invariant a contract input's signature has to satisfy
