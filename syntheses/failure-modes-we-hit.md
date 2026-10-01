---
pageType: synthesis
id: synthesis.failure-modes-we-hit
description: Concrete failure modes found by auditing bch-bot and the Omarchy plugin in production — silent-fallback bugs, gates that pass while printing errors, fabricated constants, and unit/label mismatches. Written from fixes that shipped, 2026-10-01.
sourceUrl: internal/synthesis
---

# Failure modes we actually shipped (and then removed)

Most security writing is about attacks. This page is about bugs that passed code
review, passed a security gate, and shipped — because each one *looked* correct
at the call site. Every entry here is a bug that was found by auditing or by
adversarial testing in September–October 2026, then fixed and pinned with a
test. They generalise to any wallet, CLI, or shell-plugin integration.

The recurring theme: **a fallback that returns a plausible value instead of
failing is worse than a crash**, because the crash is visible and the plausible
value is not.

---

## 1. `findIndex(...) ? 0` — a silent wrong key

```js
const idx = addrs.findIndex((a) => a.address === u.address);
const path = { change: 0, index: idx >= 0 ? idx : 0 };   // signs with /0/0
```

An address outside the gap-limit window is not ours, so `findIndex` returns `-1`,
and the fallback turns that into index 0. The transaction is then signed with
the key for `/0/0` — a *different* key from the one the address was derived
from. Depending on which other UTXOs are in range, this either fails to
validate (harmless) or spends against a different UTXO than the fee arithmetic
assumed.

Three variants existed in the same codebase:

| Variant | Produced |
|---|---|
| `index: idx >= 0 ? idx : 0` | index 0 — wrong key |
| `index: addrs.findIndex(...)` (no fallback) | index **-1**, which flows into the derivation |
| `change: recvIdx >= 0 ? 0 : 1, index: recvIdx >= 0 ? recvIdx : chgIdx` | a **receive**-chain UTXO signed with a **change**-chain key |

The third is the one to look for. "Not a receive address" does not imply "is a
change address" — the address may be in neither list, and that branch happily
manufactures a plausible path.

**Rule:** resolve the address, or fail. There is no correct fallback. The error
message should name the real remedy — *widen the gap limit* — because a wrong
index is a wrong key, and guessing is the failure being removed.

> Generalisation: any `?? 0`, `|| 0`, `? 0`, `|| default`, or `arr[0]` in a path
> from external data to a key, an account number, an output index, or a
> destination is worth reading twice.

---

## 2. A defaulted argument that silently picks the wrong network

```js
export async function connect(network = 'chipnet') {
  const list = SERVERS[network] || SERVERS.chipnet;   // AND a fallback
}
```

One script out of ten called `connect()` with no argument. Every other script
passed `w.network`. On a mainnet wallet that meant:

- addresses derived correctly from the seed (so nothing throws),
- scripthashes queried against **chipnet**,
- no UTXOs found,
- reported as *"no BCH UTXOs available"* — which reads like an empty wallet,
  not like a wrong-network lookup.

The double fallback is the real lesson: `SERVERS[network] || SERVERS.chipnet`
also swallows a **typo**. A misspelt network name silently becomes chipnet.

**Rule:** for anything that selects *where* data comes from, require the argument
and reject unknown values by name. A single choke point covers every caller, so
the rule holds for the whole codebase and a new script cannot reintroduce the
bug by forgetting an argument.

Verified: `undefined`, `''` and `null` all throw; `'miannet'` throws naming the
valid set; `'mainnet'` still connects.

---

## 3. A gate that printed an error and then reported success

```bash
IN_PANEL=$(grep -coE "..." ./BchWalletPanel.qml 2>/dev/null || echo 0)
if [ "$IN_PANEL" -lt "$TOTAL" ]; then   # never runs
```

`grep -c` prints `0` **and exits 1** when it matches nothing. The `|| echo 0`
fallback therefore appends a *second* `0`, and `IN_PANEL` becomes the two-line
string `"0\n0"`. `[ "0\n0" -lt 1 ]` raises `integer expression expected` on
stderr, the `if` takes the false branch, and the check reports success.

Reproduced on the unfixed gate: a sensitive command sat outside the panel,
`TOTAL=1`, `IN_PANEL="0\n0"`, and the gate exited 0 — **exactly inverted** from
its intent, on exactly the case it existed to catch.

**Rule:** a check that emits an error and then passes is worse than one that
fails, because it destroys trust in the whole gate. Fix count sources so the
substitution is always exactly one number:

```bash
count() { grep -coE "$1" $2 2>/dev/null | awk -F: '{s+=$NF} END {print s+0}' }
```

`awk` prints one number whether or not grep matched. **Audit every `|| echo 0`
and `|| true`** — the ones used for string-emptiness tests are fine; the one used
in arithmetic is a live bug.

---

## 4. Text matching cannot enforce a semantic invariant

The gate above greps for `["bch-bot", "sweep"]` to detect a dangerous call. This
defeats it:

```qml
property var verb: ["sw" + "eep"]
run(["bch-bot", root.verb], "balance", ["BCH_CONFIRM=yes"])
```

The resolved argv is a real `env BCH_CONFIRM=yes bch-bot sweep`. The `kind`
argument is a *display label* the gate never inspects, and string concatenation
defeats every literal pattern. **No amount of additional grepping fixes this** —
it is a property of matching text in a dynamic language.

**Rule:** stop describing the call and start constraining the capability. If
process execution is confined to one audited file, and that file's argv is
asserted to be the read-only command, then a split string has nowhere to run
from.

> The intermediate attempt — *forbid `Process` outside the panel* — was wrong
> too, because the balance widget legitimately runs `bch-bot balance`. The rule
> has to distinguish "runs a read-only command" from "can run anything", and
> that distinction is a decision, not a grep.

---

## 5. Fabricated constants that look real

```js
const ARTIFACT_FINGERPRINTS = {
  StabilityPool: 'e2136acf6f1cbb1a0fce6b1bece6b1bece6b1bece',  // real prefix, invented tail
  StabilityPoolSidecar: '4bb467b6...',                        // truncation
};
```

Five placeholders and one fabrication. The fabricated one is the dangerous
shape: a **real-looking prefix followed by an invented tail** is
indistinguishable from a real hash to a reader, so nobody challenges it. A
fingerprint check on it either always fails or — compared by prefix — silently
accepts a substituted contract.

The constant also had **zero readers**, so nothing would ever have caught the
wrong value.

**Rules:**
- **Read constants from the published artifact, never recall them.** These six
  came out of `npm pack @paryonusd/contracts@1.0.0`, read from each artifact
  file. The real `StabilityPool` value is
  `e2136acf16013e341012ea1fb48cbb59972c2d0f8db6d9bf3c15f079be2607f3` — the
  fabricated one had kept the right 8-character prefix and invented the rest.
- **A fingerprint check must fail closed.** "We have never seen this contract"
  is precisely the case a substituted contract arrives in.
- **Test the negatives.** A one-character difference and a bare prefix must both
  be refused — that is exactly what the truncations would have produced had
  anything compared against them.
- A constant with no callers is either dead code or an unfinished check. Find
  out which before trusting it.

---

## 6. A UI label that disagreed with the code it drives

The panel's send field was labelled **"Amount (BCH)"** with placeholder `0.001`.
The CLI did `BigInt(satsArg)`.

- Typing what the UI said → `error: Cannot convert 0.001 to a BigInt`, surfaced
  to the user as `bch-bot exited 1`. **The primary flow could not send at all.**
- The natural "fix" was worse than the bug: type `1000` to satisfy the parser,
  and you send 1000 sats — **a hundredth of what you meant** — after reading a
  preview that called it `1000 BCH`.

The preview-then-confirm structure was intact and **passed its own gate**. The
number inside it was in the wrong unit. A safety property can pass its check
while being wrong.

**Rule:** the unit is part of the contract, and it must be pinned by a test.
Nothing in that suite asserted what unit `send` took, which is how the mismatch
survived. `0.001` → 100000 sats and `1000` → 1000 sats are now both asserted.

And the parse fix had its own hole on the first attempt: `BigInt("0x10")` is
`16n`, `"0b101"` is `5n`, `"1e3"` is `1000n` — so `0x10` would have quietly become
16 sats. **The new test caught it on the first run.** Match the shape
explicitly before converting.

---

## 7. "Service fee: 0.00000000" on a trade that charges 10bps

The swap preview hardcoded a zero service fee. The router charges its fee at
*build* time, so a quote genuinely carries no fee figure — but printing `0`
asserts the operation is free. A flat zero on a value-moving operation is a
false statement, not a neutral default.

**Rule:** when a number is genuinely unavailable, say so. `"charged at
execution"` is honest; `0.00000000` is not.

---

## 8. Consuming a token to save a few hundred sats

`selectSweepCandidates` included token-bearing UTXOs under 2000 sats, commented
*"Sweep if user opts in (always sweep for now)"*. Spending a token-bearing UTXO
**consumes the CashToken with it**, and the sweep's single BCH change output
does not preserve it. The token was destroyed outright, irreversibly, gated only
by the global `BCH_CONFIRM`.

The file's own header promised the opposite — *"NEVER sweeps UTXOs that hold
tokens"* — and a third comment gave a different threshold again. Three sources of
truth, the code agreeing with none of them.

**Rule:** a fee optimisation must never destroy an asset. Exclude token-bearing
UTXOs unconditionally, and **report what you skipped** — a silently skipped
token is indistinguishable from a wallet that holds none, which is the same
under-reporting failure the `/7/` chain has.

---

## 9. Checking the router's self-report instead of the bytes

The swap gate compared `quote.expectedOutput` to `build.expectedOutput`. Both
numbers come from the router, so the check proved only that the router was
internally consistent. A router that redirects the recipient output can return a
perfectly matching pair — and the transaction gets signed under `SIGHASH_ALL`.

The gate's own comment claimed it was *"the last gate before a signature
exists"* while validating a self-reported figure.

**Rule:** the thing that gets signed is the thing that has to be read. Decode
the unsigned transaction and assert:

- every output pays an address we control, **or** is a small plain output we
  account for as a fee;
- at least one output is ours (otherwise "every unknown output is a small fee"
  has no anchor);
- a **token-shaped** output is never accepted as a fee.

A plain P2PKH output is 25 bytes and cannot silently carry a CashToken, so
output length is a reliable way to tell the router's fee from an asset leaving
the wallet. The fee ceiling should be derived from what the build *reported*,
not hardcoded in bps that can drift.

**The follow-on:** `SIGHASH_ALL` binds the signature to the transaction, but
nothing bound *which* transaction to the caller's intent. The router names
inputs in `inputsToSign`; the caller independently knows which UTXOs it funded.
Those must be compared. Do it in **both byte orders** — routers report the
internal txid order, callers hold display order — and compare `vout` numerically,
because Electrum returns a string and libauth returns a BigInt. A strict
`BigInt` comparison would fail on the honest case and teach you to disable the
check.

---

## 10. A test that passes for the wrong reason

The redirect test hardcoded an attacker address. It had a **bad checksum**, so
`cashAddressToLockingBytecode` returned `undefined` and the gate refused with
*"output 0 is neither one of our addresses nor a plain P2PKH output: it has 0
bytes, so it may carry a token"*.

The test went green. It never exercised the attack.

**Rule:** when a test goes green unexpectedly, read the failure message before
the pass. Derive test addresses from real keys
(`encodeLockingBytecodeP2pkh` → `lockingBytecodeToCashAddress`) rather than
typing them. **A test that passes for the wrong reason is worse than no test**,
because it buys false confidence and occupies the slot where a real test should
go.

---

## 11. A header that documents a safety property the code does not have

`sweep.mjs`'s header claimed *"NEVER sweeps UTXOs that hold tokens"*. The code
swept them. A comment elsewhere gave a third, different threshold.

**Rule:** when a comment asserts a safety invariant, make it a check. The
plugin's gate is exactly that idea — the QML/CLI boundary encoded as failing
checks rather than README paragraphs. A rule written in prose is out-competed by
whatever the surrounding code already does; a rule that fails the build cannot
be. The same applies in-file: if a header says "never X", a test must fail when X
happens.

---

## 12. Comments that describe the wrong threshold

`buildSwapExecutionTx` was dead code, ~94 lines, referencing an identifier
(`PayoutRuleType`) that was never imported and does not exist in the dependency.
It only failed because a *different* call threw first and masked the
`ReferenceError`.

**Rule:** "provably dead" and "latently broken" are different claims and need
different evidence. The presence of a placeholder key
(`new Uint8Array(32), // placeholder`) next to live signing code is a reliable
tell that a function was never finished and should be **deleted**, not repaired
— and that a future agent must not "fix" it by inventing a value.

---

## 13. A test suite that cannot fail

43 assertions asserted on values constructed inline in the same block, testing
no function in `lib/`. One defined the function it was testing *inside* the
test. Another asserted `typeof x === 'number'` on a value the test had just
computed. The file imported nothing from the codebase.

**Rule:** a test that cannot fail under **any** defect in the code is not
coverage, it is decoration. It is the largest single block of false confidence
in a suite, because the passing count makes the suite look healthier than it is.
Worth knowing that some runners print `RESULT: n` and others print
`PASSED n, FAILED n` — grepping one of them under-reports the real total.

---

## 14. Constants that drift from the code that reads them

`WALLET_DIR` is resolved at module load from `process.env.BCH_WALLET_DIR`, while
`encrypt-wallet.mjs` re-derives the same paths independently. Two sources of
truth for the path of the file holding the seed. If they ever disagree,
`encrypt-wallet` rewrites a different file than the CLI reads — and the wallet
appears to have lost its funds.

**Rule:** resolve configuration once, in one module, and have every consumer
import it. A path computed twice is a coin flip.

---

## The through-line

Every one of these is a case where the code **looked** like it was doing the
right thing:

| Bug | What it looked like |
|---|---|
| `findIndex ? 0` | a defensive default |
| `connect()` default | a sensible convenience |
| `grep -c \|\| echo 0` | error tolerance |
| literal greps | an enforced boundary |
| `"e2136acf..."` | a real fingerprint |
| `"Amount (BCH)"` | a correct label |
| `"Service fee 0.00000000"` | a default |
| `"always sweep for now"` | a policy decision |
| quote vs build match | a verification |
| a green test | a passing test |

The defence is the same in every case: **push the rule down the mechanism
hierarchy** — type system → lint that fails CI → banned API → runtime check →
prose. A value-moving wallet should get its rules from the first four, and treat
prose as documentation of what the mechanisms already enforce. Most of the bugs
on this page existed precisely because a rule was living only in prose, or in a
comment that had drifted.
