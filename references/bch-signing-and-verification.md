---
pageType: reference
id: reference.bch-signing-and-verification
description: Why a structurally valid Bitcoin Cash signature can still be rejected by the network — four libauth traps (byte-typed preimage fields, SHA256d before signing, complete prevout lists, the silently dropped correspondingOutput) and the only reliable check: verify the signature cryptographically.
sourceUrl: https://github.com/bitauth/libauth
---

# BCH signing, and how to know it actually works

> Every trap on this page produces a **plausible but invalid** signature rather
> than an error. A correctly sized scriptSig full of bytes that no verifier
> accepts looks exactly like a working one. The only thing that catches this is
> checking the signature cryptographically.

## The problem this solves

When you sign a transaction someone else assembled — which is what a
[Cauldron Router](entity.riften-router.md) swap is — you produce a
scriptSig, re-encode, and hope. Nothing in libauth, in the encoder, or in a
successful `encodeTransaction` call tells you whether the digest you signed was
the digest a verifier will recompute. Four separate mistakes all yield a
perfectly well-formed 100-byte scriptSig and a transaction the network rejects.

This is recorded here because each trap cost real debugging time and none of
them raise an error.

## Trap 1 — preimage fields are bytes, not numbers

libauth assembles the signing serialization with a reducer that reads
`.length` off every element:

```js
const flattenBinArray = (array) => {
  const totalLength = array.reduce((total, bin) => total + bin.length, 0);
  ...
}
```

So `signingSerializationType` and `forkId` must be `Uint8Array`s:

```js
// WRONG — a number has no .length; it is silently dropped from the preimage
generateSigningSerializationBCH(ctx, { coveredBytecode, signingSerializationType: 0x41 })

// RIGHT
generateSigningSerializationBCH(ctx, {
  coveredBytecode,
  signingSerializationType: new Uint8Array([0x41]),
})
```

Note the value: `0x41` for a plain BCH spend (`forkId 0x40 | allOutputs 0x01`),
`0x61` for one that carries a CashToken (adds `utxos 0x20`).

## Trap 2 — the serialization is the preimage, not the digest

libauth's Schnorr signer pads its `messageHash` argument into exactly 32
bytes. The signing serialization is longer, so passing it raw overflows:

```
RangeError: offset is out of bounds
  at cloneAndPad (secp256k1.js)
```

Double-SHA256 it first — the same step `bch-bot`'s
`cashscript.mjs::computeCovenantSighash` performs:

```js
const digest = hash256(serialization);                       // SHA256d
const signature = secp256k1.signMessageHashSchnorr(privateKey, digest);
```

Argument order is `(privateKey, messageHash)`.

## Trap 3 — prevouts must cover every input

The `utxos` sighash flag commits to the previous output of **every** input in
the transaction, not just the one being signed. libauth reads those from
`sourceOutputs`, index-aligned to the inputs.

For a router swap this is unavoidable: 29 inputs, and the wallet's own input
sits at index 28. Synthesising placeholders for the pool inputs produces a
preimage that no verifier accepts, so **fail instead**:

```js
if (!Array.isArray(routerPrevouts) || routerPrevouts.length < decoded.inputs.length) {
  throw new Error('source_outputs are required ...');
}
```

A clear error beats a signature that is quietly wrong.

## Trap 4 — `correspondingOutput` is dropped in silence

This is the one that cost the most time, because nothing failed.

`generateSigningSerializationComponentsBCH` returns `correspondingOutput` only
when `inputIndex` falls **inside the output list**:

```js
correspondingOutput: context.inputIndex < context.transaction.outputs.length
    ? encodeTransactionOutput(context.transaction.outputs[context.inputIndex])
    : undefined,
```

A router swap routinely has more inputs than outputs, so for the wallet's own
input this is `undefined`. `flattenBinArray` then drops it without complaint —
and that field is where the pubkey hash lives. The signature is produced, the
transaction encodes, and the pubkey hash is simply **not in the digest** the
network recomputes.

Supply it explicitly:

```js
const coveredBytecode = prevouts[index].lockingBytecode;      // the P2PKH lock script
const correspondingOutput = { lockingBytecode: coveredBytecode, valueSatoshis };
generateSigningSerializationBCH(
  { transaction, sourceOutputs, inputIndex, input, correspondingOutput },
  { coveredBytecode, signingSerializationType }
);
```

For a P2PKH spend the "corresponding output" is the locking script the
signature commits to.

## A rejected fix that looks like a fix

Padding `transaction.outputs` until the index lands in range silences the
symptom — and corrupts `hashOutputs`, which changes the digest and invalidates
the signature. It is the worst of the options: green tests, broken
transactions. Reject any fix that mutates the assembler's outputs.

## Verifying instead of assuming

`signMessageHashSchnorr` succeeding means nothing. Check the signature:

```js
const decoded = decodeTransactionBCH(Buffer.from(txHex, 'hex'));
const scriptSig = decoded.inputs[myIndex].unlockingBytecode;
const signature = scriptSig.slice(1, 65);              // skip the push byte
const publicKey = scriptSig.slice(67, 67 + 33);

// Rebuild the digest INDEPENDENTLY, as a verifier would, then check it.
const serialization = generateSigningSerializationBCH(
  { transaction, sourceOutputs, inputIndex, input, correspondingOutput },
  { coveredBytecode, signingSerializationType: new Uint8Array([0x41]) }
);
const valid = secp256k1.verifySignatureSchnorr(signature, publicKey, hash256(serialization));
```

Rebuilding the digest in the test rather than trusting the signer's own
internals is what gives the check its value. Both `bch-bot` test files
(`test-router.mjs`, `test-external-signing.mjs`) do this, and
`test-external-signing.mjs` fails if verification is ever skipped.

Shape assertions are worth keeping but they are not sufficient:

- scriptSig length is 100 for P2PKH (`push 65 + sig + push 33 + pubkey`)
- sighash byte at offset 65 is `0x41` (or `0x61` with a token)
- the embedded pubkey equals the one derived from the signing key
- inputs not in `inputs_to_sign` are byte-identical to what the assembler
  produced — a partial signer must not touch the DEX's own signatures

## Why partial signing is different

`generateTransaction` builds a whole transaction from inputs and outputs; it
cannot sign one in place. For a foreign-assembled transaction the flow is:

1. `decodeTransactionBCH(hex)`
2. for each index in `inputs_to_sign`: build the digest, Schnorr-sign it
3. write `<push sig> <push pubkey>` into that input's `unlockingBytecode`
4. `encodeTransaction` the whole thing again

Every other byte must survive unchanged. The `utxos` flag means the digest
depends on the pool inputs too, so "leave them alone" and "they are part of
what you signed" are both true at once — which is exactly why `source_outputs`
has to be complete and real rather than stubbed.

## Checklist before signing a foreign transaction

- [ ] `source_outputs` covers **every** input, with real locking scripts
- [ ] `signingSerializationType` and `forkId` are `Uint8Array`s
- [ ] the serialization is SHA256d'd before signing
- [ ] `correspondingOutput` is supplied explicitly
- [ ] the transaction's outputs are **unmodified**
- [ ] inputs outside `inputs_to_sign` are byte-identical afterwards
- [ ] the signature verifies against an independently rebuilt digest
- [ ] the quote the user agreed to still matches the built transaction

## Related

- [Riften Labs Cauldron Router](entity.riften-router.md) — the service that
  produces the transaction being signed here
- [libauth](libauth.md) — the library whose encoder these traps belong to
- [BCH wallet threat model](security/wallet-threat-model.md) — key storage and
  operational threats
