---
pageType: reference
id: reference.gates-that-pass-when-they-should-fail
description: A safety gate that returned the right answer to the wrong question, and three client defects that made a stale route look healthy. How a validation check can be green, on schedule, and completely useless — and the three questions that catch it.
sourceUrl: https://docs.riftenlabs.com/cauldron/swap/
---

# Gates that pass when they should fail

The swap printed this immediately before broadcasting, three times in a row:

```
[3/4] building the unsigned swap...
      we sign input(s) [12,13]
      pool inputs verified unspent (12/12)
[4/4] broadcasting...
      rejected: Missing inputs
```

Twelve of twelve pool inputs verified live, and every one of them was already
consumed. This is the most expensive shape of bug there is: **a check that
reports success while the thing it guards is broken.**

Luke's response — *swaps are obviously happening on Cauldron right now, how do
other wallets solve this?* — is what found it. I had assumed a dead route. He
pointed out the protocol was live, which meant the fault was ours.

---

## Defect 1: the right question, the wrong subject

The gate asked:

```js
scriptHasUnspent(lockingBytecode)   // "does this covenant have ANY live coin?"
```

The gate needed to ask:

```js
outpointIsUnspent(lockingBytecode, vout, txid)   // "is THIS position live?"
```

A competing swap re-creates the covenant at a **new position**. The lock keeps
live outputs forever, so the lock-level question is permanently true for an
active pool — and permanently true for a position that no longer exists.

Measured on parent `fd02de7d…1138`, 56 outputs, **all 56 spent**:

```
v 0  gate says unspent   exact outpoint: SPENT
v 1  gate says unspent   exact outpoint: SPENT
v 2  gate says unspent   exact outpoint: SPENT
...
v 6  gate says unspent   exact outpoint: SPENT
```

Meanwhile the same covenant locks report live coins from *other* transactions:

```
h971235 v0   0f5e93ac…   1717810 sats
h971247 v43  ebcf3aee…  11324336 sats
h971247 v44  ebcf3aee…  11355884 sats
h971262 v2   6038da57…   5672160 sats
```

**The pool was alive and trading. Our position in it was not.** The gate had no
way to hold those two facts apart.

> **A check scoped to a container passes whenever the container is non-empty.**
> If the thing you care about is one *element* of a changing collection, query
> the element — txid **and** vout — not the collection.

## Defect 2: the gate read through a node that cannot see the data

```js
const checkClient = await connect(w.network);   // plain Fulcrum
```

Cauldron pool outputs are **p2sh32 covenants**, and Fulcrum does not index them.
Every parent fetch returned null, every pool input read as "parent unknown", and
the gate degraded to reporting that it could read nothing — on every run, silently.

The repo already documented this exact hazard elsewhere and still got it wrong
here. The fix is `connectToken()`, the token-aware client. Parents then resolve
**13/13**.

## Defect 3: the one case that mattered was exempted

```js
} else if (stale.length === poolInputCount) {
  // "every pool input failed to read — transport problem, proceed and let the
  //  broadcast decide"
```

That is **the drained-route case**, commented as if it were the unreadable case.
A real transport failure lands in `unreadable`, never in `stale`. So the single
most important verdict the gate could return was the one it was written to
suppress.

The two are now separate branches. An all-stale route refuses to sign:

```
REFUSING TO SIGN -- all 12 pool input(s) are already spent.
```

…in ~38 seconds, instead of signing and losing three broadcasts.

---

## Node capability is not optional context

`outpointIsUnspent` asks **one** node, on purpose. Two would be wrong:

| node | unspent outputs for a live covenant lock |
|---|---|
| `rostrum.cauldron.quest` | **9** |
| `bch.imaginary.cash` | 0 |
| `cashnode.bch.ninja` | 0 |
| `fulcrum.jettscythe.xyz` | 0 |

Only Rostrum indexes p2sh32. The others return 0 not because coins are gone but
because they cannot see the script. **A "two nodes agree" rule applied to that
set marks every live route dead** — the mirror image of the original bug, and
just as wrong.

Capability has to be a property of the *check*, not an assumption:

> Before treating a negative as evidence, establish that the responder can
> return a positive for this data at all. Otherwise "no" means "I cannot see".

## The three questions

These are what would have caught it, in order of cost:

1. **What question does this check actually ask?** Write it as a sentence. If
   the sentence doesn't mention the specific thing you're protecting, the check
   is too coarse.
2. **Can this source return a positive for this data?** Feed it a known-good
   live example. A source that only ever answers "no" is not measuring.
3. **What happens on the *all-negative* result?** Trace that branch on paper.
   The worst bugs live there, because it is the branch nobody tests.

## The debugging lesson underneath

This took three days and produced six wrong explanations — contention,
nonexistent prevouts, unsigned transaction, invalid output index, a
covenant-blind node, and "the pools are contested". Every one came from reading
a tool's output as a verdict.

Two were **transport** defects wearing the costume of facts about the chain: a
client that collapses JSON-RPC errors into `{}`, and a check that asked a
covenant-blind node. In both cases the answer was available the whole time and
the instrument was what failed.

> When an answer is surprising, the first suspect is the code that delivered it,
> not the thing it describes. A green check is a claim, not evidence — and a
> check that has never been observed to fail has not been tested.

## The rules are now encoded, not written down

The three questions above were prose. Prose is the weakest layer of the
mechanism hierarchy — an agent reads the surrounding code and copies that, so a
documented rule loses to a stronger habit in the file next door. Two lints now
fail CI, and each is scoped to exactly one question from this page.

| Lint | Encodes | Fails on |
|---|---|---|
| `lint-check-questions.mjs` | *What question does this check ask?* | a function named as an exact check (`isUnspent`, `hasUnspentOutput`, `isValid`) with no txid/vout in scope |
| `lint-covenant-node.mjs` | *Can this source return a positive?* | `connect()` on a covenant-aware path, where the node cannot index p2sh32 |
| `test-lints-catch-their-own-bugs.mjs` | — | either lint failing to catch the defect it was written for |

That last one exists because of how this went. **Both lints were broken while
they were being written, and both reported `clean` on the exact code they
existed to reject:**

- `(?:is|has)(?:Unspent|…)` can never match `scriptHasUnspent` — there is no
  word boundary between the `h` and the `U`, because they are one word.
- The scope walk began brace-counting on the line holding the parameter list's
  closing paren, which is the same line as the body's opening brace, so every
  function measured one line long and no scope ever contained a `connect()` call.

A lint that has only ever run against correct code is not a lint. The proof
suite reintroduces each real defect into a scratch copy and requires the lint to
fail, then requires it to pass on the fixed code and on the legitimate P2PKH
neighbours — 9 assertions, because a rule that flags honest code teaches people
to disable it.

`lint-covenant-node` is deliberately narrow. It does not flag `runAttempt()` in
`swap.mjs`, which reads token UTXOs from the wallet's own P2PKH addresses
through `connect()`: that is correct, and a broader rule would have trained
everyone to bypass it.

## Related

- [`./cauldron-k-invariant.md`](./cauldron-k-invariant.md) — the two rules a
  valid Cauldron swap must satisfy, and the byte-order trap
- [`../security/dex-swap-integration.md`](../security/dex-swap-integration.md) —
  the live-swap addendum
