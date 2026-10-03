---
pageType: reference
id: reference.security.pre-signing-invariant
description: The class of bug where a value-moving path depends on an identifier it never imported, or on a check that only inspects the peer's own self-report. Three real instances in bch-bot (send.mjs and round-trip.mjs dead on arrival; the swap gate never reads the value of our own receive output) plus the CI shape that lets all of them through a green suite.
sourceUrl: internal/audit
---

# Pre-signing invariants: when the thing that gets signed is never checked

*Written 2026-10-02 from a direct audit of the live code at `~/bch-src` on the
Omarchy box (git `d63bda9`), run read-only. This page does not repeat
[`cashtokens.md`](cashtokens.md), [`script-and-signing.md`](script-and-signing.md)
or [`dex-swap-integration.md`](dex-swap-integration.md). It documents three
findings those pages do not contain, and the general shape they belong to.*

**The through-line.** Every bug on
[`../syntheses/failure-modes-we-hit.md`](../syntheses/failure-modes-we-hit.md)
was a check that *looked* like it was enforcing something. This page is about the
two remaining ways that happens when the check is not merely wrong but **not
present at all**: the code that would run it references an identifier that does
not exist, and the gate that would catch it reads the counterparty's own
self-report instead of the bytes.

---

## 1. `send` — the wallet's primary command — cannot run at all

**Severity: high. Impact: availability, not loss. Exploitable today: yes, by
anyone who runs the command.**

`scripts/send.mjs` calls two functions it never imports:

```js
// scripts/send.mjs:10-12
import { connect, scripthashForAddress, listUnspent } from '../lib/network.mjs';
import { loadWallet, loadHdNode, resolveAddressPath, newChangeAddress } from '../lib/wallet.mjs';
//                                             ^ deriveReceivingAddresses absent
//                                                        deriveChangeAddresses absent

// scripts/send.mjs:26-29
const addrs = [
  ...deriveReceivingAddresses(20).map((a) => ({ ...a, chain: 'recv' })),
  ...deriveChangeAddresses(20).map((a) => ({ ...a, chain: 'change' })),
];
```

ES modules have no dynamic scoping, so the reference resolves to nothing at call
time. The command dies at the first UTXO scan:

```
$ env BCH_WALLET_DIR=<mainnet wallet> node scripts/send.mjs <addr> 0.001
[wallet] WARNING: wallet.json is plaintext (v1). ...
network: mainnet
error: deriveReceivingAddresses is not defined
exit=1
```

**Root cause is identifiable to the exact commit.** `53b3507` ("Make three
read-only scripts see change addresses, and stop dry runs from mutating wallet
state") rewrote the scan to cover both chains. The diff shows the new calls; the
same commit's import line was not updated:

```
$ git show 53b3507 -- scripts/send.mjs | grep -E '^\+|^-'
-  const addrs = deriveReceivingAddresses(20);
+  const addrs = [
+    ...deriveReceivingAddresses(20).map((a) => ({ ...a, chain: 'recv' })),
+    ...deriveChangeAddresses(20).map((a) => ({ ...a, chain: 'change' })),
+  ];
```

This is a **fix that shipped broken**, and it is the exact failure mode
`failure-modes-we-hit.md` §13 predicts — except that here the test suite is not
merely vacuous, it is **absent for this file**.

### The same bug exists in `round-trip.mjs`

```
$ node scripts/round-trip.mjs
error: deriveReceivingAddresses is not defined
ReferenceError: deriveReceivingAddresses is not defined
    at main (file:///home/luke/bch-src/scripts/round-trip.mjs:55:17)
```

Same class, same shape, second file. A sweep of every non-test entry point
(`for f in scripts/*.mjs`, throwaway wallet dir, no broadcast env) finds exactly
these two and no others:

```
add-liquidity   <no ReferenceError>   address     <no ReferenceError>
balance         <no ReferenceError>   create-wallet <no ReferenceError>
encrypt-wallet  <no ReferenceError>   history     <no ReferenceError>
send-token      <no ReferenceError>   stake       <no ReferenceError>
swap            <no ReferenceError>   sweep        <no ReferenceError>
utxos           <no ReferenceError>
```

**Why three gates missed it.** All three are real gates, and all three are
blind to this specific defect:

| Gate | What it checks | Why it passes |
|---|---|---|
| `node scripts/lint-unused.mjs` | files importing a binding they never use | This is the *inverse*: an identifier used but never imported. Not in scope. Reported `clean (44 files checked, 0 unused imports)`. |
| `npm test` → `test-send-amount.mjs` | the amount-parsing rule | It **re-implements** the rule rather than calling `send.mjs`: `// This asserts the conversion the script uses. It cannot call send.mjs's main()`. It exercises a copy, so the real script is never loaded. |
| CI (`.github/`) | runs the two above | Inherits both blind spots. |

This is `failure-modes-we-hit.md` §13 in a new costume. The rule generalises:

> **A test that re-implements the code under test cannot fail when the code under
> test breaks.** `test-send-amount.mjs` contains the comment explaining why it
> cannot call `send.mjs` — and treats that as acceptable. The fix is a smoke test
> that *imports* each entry point, not a second implementation of its rules.
>
> **`node --check` does not catch this either.** It parses syntax only and never
> resolves identifiers in scope. Only actually executing the module — or a
> real type-checker / `eslint` `no-undef` rule — surfaces a `ReferenceError`.

**Fix (two lines):**

```js
// scripts/send.mjs:12
import {
  loadWallet, loadHdNode, resolveAddressPath, newChangeAddress,
  deriveReceivingAddresses, deriveChangeAddresses,   // <-- add
} from '../lib/wallet.mjs';
```

`round-trip.mjs` needs the same two names added to its `lib/wallet.mjs` import.

**Status note.** The working tree also carries an uncommitted edit to
`scripts/swap.mjs` (the `request(m, [txid, false])` → variadic fix) plus an
untracked `probe-poolcheck.mjs`. Both predate this audit; nothing in this audit
modified the repo.

---

## 2. The swap gate verifies *where* our money goes, never *how much*

**Severity: high. Impact: direct loss of the traded amount. Exploitable today:
yes, by a malicious or compromised router. This is the highest-value finding on
this page.**

`verifyTransactionOutputs()` in `lib/router.mjs` is the local check that stands
between the router and a signature. It is genuinely good at what it does: it
decodes the unsigned transaction, byte-compares every output against the
addresses we supplied, refuses unrecognised shapes, bounds foreign P2PKH leakage
against a fee ceiling, checks token categories against the sell asset, and
enforces the pool count. See [`dex-swap-integration.md`](dex-swap-integration.md)
for the design rationale.

**What it never does is compare the value of the output paying our receive
address against `expected_output` or against the user's slippage floor.** The
parameter `maxSellValueSats` looks like it would, and is accepted and documented —
but is dead:

```js
// lib/router.mjs:304-306 — accepted, documented, never read in the body
  // The satoshi value the user is selling, when selling plain BCH. The total
  // routed into pool covenants cannot exceed it. Null skips that comparison.
  maxSellValueSats = null,
```

`grep -n maxSellValueSats lib/router.mjs` returns only the destructuring
declaration at line 306. Every ownership check is `isOurs`, a **byte comparison on
the locking script** — never on the value:

```js
// lib/router.mjs:388
const ours = allowed.some((b) => sameBytes(b, bytes));
```

And the two checks that *do* compare an amount compare router-to-router, which
`failure-modes-we-hit.md` §9 already names as insufficient:

```js
// lib/router.mjs:255 — quote's outputAmount vs build's expectedOutput, both router-supplied
// lib/router.mjs:261 — build's own expectedOutput vs the user's minOutput
```

**Verified by building real transactions and running the project's own gate**
(`/tmp/bchprobe/probe.mjs`, importing the repo's `lib/router.mjs` unmodified):

```
A honest build            -> ok = true   (expected true)
B receive output = 1      -> ok = true   <== UNDER-DELIVERY PASSES THE GATE
C same, minOutput=36000   -> ok = true   (minOutput not consulted)
D redirect to attacker    -> ok = false  (refused, correct)
E quote-vs-build          -> ok = true   (compares router numbers to router numbers)
```

Case B is the attack. The router reports `expected_output = 36141` (and satisfies
`min_output = 36000` in the *reported* figure), assembles a transaction that pays
**1 base unit** to our receive address, and routes the rest to the pools. Every
gate passes:

- the receive output **is** ours, so it is never value-checked;
- the change output **is** ours, so it is never value-checked;
- whatever went missing is inside the **pool covenants**, which are excluded from
  the conservation check because *"covenant output values are not satoshis"*
  (`lib/router.mjs:423`) — a correct, necessary exclusion that is exactly what
  makes this hole reachable;
- `maxInputValueSats` compares only against `isSimpleP2pkh && !o.isOurs`
  (`lib/router.mjs:522-524`), and our two outputs are both ours.

The user signs under `SIGHASH_ALL` and pays full price for nothing. The
`dex-swap-integration.md` checklist asks the integrator to confirm *"the recipient
output equals the amount you asked for"* — the code does not implement that item.

**Fix.** `verifyTransactionOutputs` should take the expected amounts and check
them against the outputs it has already proved are ours:

```js
// in lib/router.mjs, after the ownership pass:
const expectedOutput = expectedOutputBaseUnits;           // build.expectedOutput
const receiveOut = outputs.find((o) => o.isOurs && sameBytes(o.lockingBytecode, receiveBytes));
if (!receiveOut) problems.push('no output pays our receive address');
else if (BigInt(receiveOut.valueSatoshis) < BigInt(expectedOutput))
  problems.push(`receive output pays ${receiveOut.valueSatoshis}, the build reported ${expectedOutput}`);
// and re-assert the floor against the BYTES, not the reported number:
if (minOutput != null && BigInt(receiveOut.valueSatoshis) < BigInt(minOutput))
  problems.push(`receive output ${receiveOut.valueSatoshis} is below the floor ${minOutput}`);
```

For a **token** swap the received asset is in the token prefix, not
`valueSatoshis`, so the comparison must read `o.tokenCategory` +
`o.valueSatoshis`-as-amount. The parser already decodes both; it just does not
compare them. **Judgement, not an established standard:** the shape of this check
is my recommendation. The requirement — *the recipient output equals the amount
you asked for* — is the operator's own published checklist.

---

## 3. A dry run is a signature, not a preview

**Severity: medium. Impact: state divergence over time, not instant loss.**

Every value-moving script signs before it checks `BCH_CONFIRM`. `send.mjs:144`
signs, then `:151` reads the env var. So does `send-token.mjs:264` → `:269`, and
`sweep.mjs:161` → `:166`.

This is correct as a design (a dry run that cannot sign is not a useful preview)
and it is not itself a bug. It becomes one in combination with `swap.mjs`'s
already-documented change-address handling, where a dry run had to be changed to
stop consuming state:

```
$ git show 53b3507
  swap.mjs called newChangeAddress() unconditionally, which derives the current
  change index, increments it, and writes state.json. ...
  (observed: 10 -> 33 across ~23 runs)
```

`swap.mjs:216-218` now derives without consuming, and only advances on a
confirmed broadcast. **But `send.mjs`, `send-token.mjs`, `sweep.mjs`,
`round-trip.mjs` and `add-liquidity.mjs` still call `newChangeAddress()`
unconditionally**, so each of their dry runs burns an index. `add-liquidity.mjs:209`
is the clearest case — it is `newChangeAddress()` before the `--broadcast` check
at `:251`.

The live state file shows the accumulated cost:

```
$ cat <wallet dir>/state.json
{ "address_index": 1, "change_index": 39 }
```

An index is 20 gap-limit slots of BCH privacy (§5 below), so 39 unused change
addresses is not itself a loss — but it is unbounded, and it is silent. **Rule:**
the change-address-consuming call belongs **after** the confirmation gate in every
script, and the gate should be checked once, at the top, rather than re-derived
per script.

---

## 4. The confirmation gate is three different flags

`BCH_CONFIRM=yes` is the moth convention and it is honoured everywhere. But
`add-liquidity.mjs` adds a second, independent gate:

```js
// scripts/add-liquidity.mjs:251
if (!opts.broadcast || process.env.BCH_CONFIRM !== 'yes') {
```

So that command needs **two** opt-ins, while every sibling needs one. Nothing is
unsafe today — it is the safe direction — but a user who has internalised "set
`BCH_CONFIRM=yes` and it broadcasts" gets a silent dry run from `add-liquidity`
instead. Two flags for one property is a convention that will eventually drift
into one flag on the wrong command. **Rule: one destructive-action gate, checked
in one place.** `swap.mjs` already shows the shape (`isBroadcast` computed once at
`:216`); the rest should follow it.

---

## 5. Unbounded derivation is available from the CLI

```js
// scripts/address.mjs:18 — no upper bound
if (args[i] === '--count') out.count = parseInt(args[++i], 10);
// :33
if (opts.count > 1) { const addrs = deriveReceivingAddresses(opts.count); ...
```

`deriveReceivingAddresses(n)` performs `n` sequential secp256k1 derivations
(`lib/wallet.mjs:256-269`) with no cap. `--count 5000000` is a local
denial-of-service against the wallet process and, more usefully to an attacker, a
way to make the host burn CPU while holding the seed in memory. It is not remote
exploitable — it requires local CLI access — so the practical severity is low,
but the fix is one line and the pattern (an unbounded user-supplied loop over key
material) deserves to not exist. **Judgement:** cap at something like 1,000 and
say so in the error.

---

## The through-line

| Finding | What it looked like |
|---|---|
| `send` uses two functions it never imported | a fix that made the scan cover both chains |
| The gate checks ownership, never amount | "every output must be ours — that is enough" |
| `maxSellValueSats` accepted, documented, never read | a parameter that documents intent |
| Dry runs burn change indices | "dry run = preview" |
| Three gates, all blind | "the suite is green, and lint is clean" |

Every one is the `failure-modes-we-hit.md` through-line wearing a different
hat: **the rule was living in prose, or in a comment, or nowhere at all.** The
defence is the same and it is already written down — push the rule down the
mechanism hierarchy, and make the check read the bytes rather than the reporter.

> The single highest-value mechanical change here is not a new check. It is making
> `npm test` **import** every entry point. That one change would have caught the
> dead `send`, the dead `round-trip`, and would have caught the `token discovery`
> bug too. Every defect on this page is invisible to a suite that never loads the
> file that ships.
