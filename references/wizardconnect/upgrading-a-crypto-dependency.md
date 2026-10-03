---
pageType: reference
id: reference.wizardconnect.upgrading-a-crypto-dependency
description: The procedure for upgrading the library a wallet signs with — capture golden signing vectors before the bump, diff the bytes after, because a green test suite cannot prove a signature is unchanged.
---

# Upgrading a crypto dependency

Written after moving `@bitauth/libauth` 3.0.0 → 3.1.0-next.2, which was required
to use WizardConnect at all. The bump touched every signing path in the wallet,
so the interesting question was not "do the tests pass" but "**how do I know the
signatures are the same?**"

## The weak evidence

A green suite. It is worth having and it is not worth much here, because a test
asserting "the signature verifies" passes just as happily for a *different but
valid* signature. Nothing in a normal test suite pins the bytes.

## The strong evidence

Sign a fixed transaction with a fixed key before the bump, and again after, and
diff the output. A throwaway seed that protects nothing, an in-process
transaction, no network:

```js
const SEED = 'abandon abandon ... about';   // well-known, holds nothing
const serialization = generateSigningSerializationBCH(context, {
  coveredBytecode, signingSerializationType: new Uint8Array([hashType]),
});
const sighash = hash256(serialization);
const signature = secp256k1.signMessageHashSchnorr(node.privateKey, sighash);
```

Captured on 3.0.0:

```
sighash    627578f120ee42c124e04840532cd78a603fdf99deda4c9102c46a72651a4923
signature  5c137cbc99c82c6932ee8e16c90491bfa44128645d525a13aa15dfba2188e8e
           ce67e0355d50e13731b40f020594e2e6901e5f84ec9c0b509746388c8102f9c9c
```

Re-run on 3.1.0-next.2: **byte-identical.** The upgrade changed nothing about
what this wallet signs. That is a claim; the diff is the proof.

Keep the script in the repo (`scripts/golden-signing-vectors.mjs`). It is 60
lines, needs no wallet and no network, and it is the check to run before
believing any future bump.

## The procedure

1. **Commit the current state first.** Not for history — so the upgrade is one
   `git revert` away. A pre-release bump of the signing library is exactly when
   you want a revert point rather than a story.
2. **Capture the vectors.** Before touching `package.json`.
3. **Install the exact version** the dependent requires, not "latest":
   `npm install @bitauth/libauth@3.1.0-next.2 --save-exact`. The repo pins exact
   versions for a reason; a pre-release range would resolve differently next
   month.
4. **Diff the vectors.** Identical or stop.
5. **Run the whole suite**, including the lints — they catch unused imports in
   whatever new code the upgrade enabled, which is the usual surprise.
6. **Confirm the new API actually exists** the thing you upgraded for. It is
   possible to bump a version successfully and still not have the export:

```js
generateSigningSerializationBCH   function   // had it
generateSigningSerializationBch   function   // what the package wanted
```

8. **Check for moved exports.** `signMessageHashSchnorr` turned out to have
   moved under a `secp256k1` namespace. Nothing failed loudly; the question is
   worth asking before something else calls the old path.

## What the bump did not change, and what it revealed

Nothing cryptographic, and the bump surfaced a real bug in code written the
night before: a P2PKH unlocking bytecode built as a single 68-byte push rather
than two. See [the signing rules](signing-rules.md).

Which is the other reason to do this properly. **A dependency bump is the
natural moment to write the tests that were missing**, because you are already
looking at the code with fresh suspicion and the marginal cost is one afternoon.

## Related

- [The signing rules](signing-rules.md)
- [`../swaps/gates-that-pass-when-they-should-fail.md`](../swaps/gates-that-pass-when-they-should-fail.md)
  — the same class of problem in validation code rather than in crypto
