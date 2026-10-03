---
pageType: synthesis
id: synthesis.failure-modes-we-hit
description: Concrete failure modes found by auditing bch-bot and the Omarchy plugin in production — silent-fallback bugs, gates that pass while printing errors, fabricated constants, unit/label mismatches, and protocol-version bugs that zero every read. Written from fixes that shipped, 2026-10-01.
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

## 15. A protocol version the servers do not implement — which zeroes every read

`lib/network.mjs` negotiated Electrum protocol `1.5`. The public mainnet nodes
(Fulcrum 2.1.2, `cashnode.bch.ninja:50004` and peers) implement `1.4` and
`1.4.3`. Asking for `1.5` does **not** produce an error. The sequence is:

- the TLS + WebSocket socket opens normally;
- `server.version` answers plausibly;
- `blockchain.scripthash.*` keeps working, so UTXO scans look healthy;
- every other `blockchain.*` call — `transaction.get`, `transaction.broadcast`,
  `cash.*` — returns an **empty object `{}`**.

The consequence is that balance reads as zero and swaps broadcast as
`Missing inputs`, with no error emitted anywhere in the process. A green log and
an empty balance are indistinguishable from a rich wallet with a bad display.

Verified by hand over a raw TLS+WebSocket session on the same host:

| Requested | Result |
|---|---|
| `["1.4","1.4.3","1.5"]` | `ERROR Unsupported protocol version` |
| `["1.4","1.4.3"]` | `["Fulcrum 2.1.2","1.4.3"]`, `transaction.get` returns real bytes |
| `["1.5"]` | `ERROR Unsupported protocol version` |

**Rule:** offer the *list* the servers implement, newest first
(`electrum: '1.4', rostrum: ['1.4','1.4.3']`), so a newer node can still select
a newer version. Pinning a single version is what created the failure. And
when a wallet reports an empty balance with no error, check the negotiated
version before anything else.

**The same page previously carried the wrong advice** — it stated that Rostrum
servers reply `"1.5"` regardless of docs and that `1.5` should be used for both
Electrum and Rostrum. That line is what the fix contradicted. Prose in a wiki is
not evidence; the raw wire is.

---

## 16. `{}` is a non-answer, not a value

Electrum's empty-object response got read three different wrong ways during one
debugging session:

- as success — a `broadcast` that returned `{}` was logged as
  `BROADCAST ACCEPTED`, when no transaction had been broadcast at all;
- as proof of absence — "the node does not have this transaction";
- as a reason to change unrelated code.

`{}` means *this call did not produce a usable answer*. The same call can return
real bytes moments later on the same host with the same session, and `{}` for a
different method shape. It is never a value to reason from.

**Rule:** before concluding anything from an empty response, re-ask over a raw
socket and print the actual reply. In this investigation every correct answer
came from a hand-rolled TLS+WebSocket session, and every wrong answer came from
reading library files or trusting a return value.

**Addendum (2026-10-02 code audit): the rule was learned and not applied.** The
non-answer rule is now correct in one place — `lib/router.mjs:583-591`
(`broadcastViaElectrum`) validates the returned txid shape and throws on anything
else. It is **still missing in five scripts**, which test only
`typeof result === 'string' && result.startsWith('Error')` and therefore print
`"broadcast": true` when the node answered `{}`:

```
scripts/send.mjs:164          scripts/sweep.mjs:182
scripts/send-token.mjs:285    scripts/add-liquidity.mjs:267
scripts/round-trip.mjs:142
```

This is the §16 rule with the same shape as §5's "a fingerprint check must fail
closed": **the lesson landed in the module that was written last and never
propagated to the four that call the same RPC.** A rule enforced in one file is
not enforced in the codebase. See
[`../security/key-custody-and-oracle.md`](../security/key-custody-and-oracle.md)
§5.

---

## 17. `request` is variadic — nesting the params array silently breaks a call

`ElectrumClient.request` is declared `request(method, ...parameters)`. Calling
`request('blockchain.transaction.get', [txid, false])` puts `[[txid, false]]` on
the wire. A method that otherwise works answers `{}`.

This one mistake cost hours, because it made a healthy node look broken: the same
transaction was `21702` bytes via `request(m, txid, false)` and `{}` via
`request(m, [txid, false])`. Every call site in the codebase used the correct
variadic form — only the ad-hoc probe was wrong — so the production path was
never affected. That is the worst case for a debugging tool: it is wrong only
when you are already relying on it.

**Rule:** `request(method, a, b)`, never `request(method, [a, b])`. When a probe
disagrees with production code, suspect the probe first.

---

## 18. A txid has two byte orders, and the wrong one looks like "no such transaction"

libauth's `outpointTransactionHash` is stored little-endian (wire order).
Electrum's `tx_hash` is the big-endian *display* form — the byte-reverse. The
two are exact reversals of each other, so a lookup with the wrong order returns
`No such mempool or blockchain transaction` for a transaction that plainly
exists. In a routed swap the parent outpoints come from libauth, so the wire
order is what the node must be asked for.

**Rule:** compute both, try both, and log which one answered. An existing
transaction reporting as unknown is a byte-order bug until proven otherwise.

---

## 19. Scanning only receiving addresses silently hides the change balance

`scripts/swap.mjs` collected funding UTXOs from `deriveReceivingAddresses(20)`
and never looked at change addresses. On the funded wallet, 856,855 of
1,657,855 sat sat on a change address, so the swap could only ever offer
801,000 sat. The wallet's *reported balance* was right; its *usable balance*
was understated by more than half, and nothing printed a warning.

`balance.mjs` and `sweep.mjs` already scanned receiving **and** change. One
script disagreeing with its two siblings is the signal.

**Rule:** when one script's numbers differ from the others, that script is the
bug. Change is money the wallet sent itself and got back — excluding it
understates spendable funds and makes later fee arithmetic wrong.

---

## 20. Signing a transaction whose pool inputs are already spent

The Riften router serves pool state that competing swaps consume. A pool
covenant that was unspent when the route was quoted may be gone by the time the
transaction is broadcast, and Bitcoin ABC rejects the result with
`the transaction was rejected by network rules. Missing inputs`.

That error reads like a malformed transaction rather than a spent input, and
discovering it after signing costs a signature plus a failed-broadcast fee. In
Bitcoin ABC `TX_MISSING_INPUTS` is set in exactly two places, both pure
UTXO-availability checks (`!HaveCoin`, `!HaveInputs`); a bad scriptSig gives
`mandatory-script-verify-flag-failed` instead. So **"Missing inputs" is never a
script or signature problem** — it is an availability problem, and an
unspent-output check is exactly the right pre-flight test.

`swap.mjs` now checks every router-owned input before signing and refuses with a
distinct exit code. Two refinements the first version got wrong:

- It reported "already spent" for a *failed read*. A node that answers `{}` to
  every `transaction.get` is a transport condition, not proof the pools are
  spent. When the check cannot read the pool inputs at all it says so and lets
  the broadcast settle the question.
- It assumed one byte order for the parent outpoint (see §18), which reported a
  healthy route as "parent unknown".

**Rule:** a safety check that cannot complete must not report a confident
negative. Distinguish *known bad* from *could not determine*, and let the
authoritative operation decide. And read the error's source before theorising
about its cause — an error string names a reason, not a subsystem.

---

## 21. A 69-byte covenant scriptSig is correct, not unsigned

A routed swap has 15 inputs and 15 outputs: 2 wallet P2PKH inputs and 13 pool
covenant inputs, plus a matching set of P2SH covenant outputs paying the pools
back. The covenant scriptSigs are 69 bytes and contain no DER signature, which
reads as "the router forgot to sign" and is not.

`POOLV0_SIZE = 6 + 20 + 43 = 69` exactly. That is CHIP-2022-05's
`<push redeem_script>` — a signature-free unlock, the `0x44` opcode plus a
68-byte push. The `swap()` ABI takes **no arguments**; the `OP_CHECKSIG` branch
of a pool contract is only for *withdraw*. A 69-byte scriptSig starting `0x44`
with no signature is the correct shape.

Note also that `bitcoinjs-lib` cannot express P2SH32 at all —
`p2sh({hash:<32 bytes>})` throws `Expected Buffer(Length: 20)`. P2SH32 work
needs libauth or cashscript.

**Rule:** before calling a script malformed, find the contract's actual
unlocking rule. A missing signature and a signature that is not required look
identical from the outside; only the spec distinguishes them.

---

## 22. Running a script directly queries a different wallet

`bch-bot` is a shim that exports `BCH_WALLET_DIR` itself. Running
`node some-script.mjs` directly skips the shim, so `lib/wallet.mjs` falls back
to `~/.bch-wallet` — on this box a different, empty wallet. Every derived
address is then wrong, `listunspent` returns nothing, and the wallet appears
empty.

This produced a convincing phantom: "the wallet holds 0 UTXOs" while
`bch-bot balance` reported three. It also produced a false alarm that the funded
addresses were not ours — they were, at index 0 and change index 19 of the
correct wallet; the scan had been reading the wrong one.

**Rule:** when running project code outside its entry point, set the
environment its entry point sets. A tool that queries a different store than
the one under test will disagree with it, and the tool is usually the one that
is wrong.

---

## 23. Token discovery that reports zero while holding tokens (and a claim I got wrong)

`outputToLibauth` in `lib/sign.mjs` passes a `token` field straight through to
libauth, and I read that as "libauth ignores it, so a token output is really a
bare P2PKH script with no CashToken prefix." I wrote that into the wiki.

**That claim was wrong, and the test that disproved it is worth more than the
claim was.** Signing a token output through the project's own path produces

```
ef53ff3501720c686780457d9affa6e60f552f5685bb6a768325f926a380ef2c8910 64 76a914…
```

which is **byte-identical** to a node-accepted mainnet ROACH output carrying 100
base units. libauth encodes the prefix exactly as specified — a 34-byte token
commitment after the `0xef` `PREFIX_TOKEN` marker, then the amount as a
CompactSize, then the locking bytecode. `token` is consumed, not dropped.

The `bad-txns-vout-tokenprefix (code 16)` rejection I attributed to this was
actually a **fee** bug (section 24). A transaction rejected for one field is not
evidence about a different field, and I let the proximity of the two guesses
become a conclusion.

**The real bug was one level down, in discovery.** `blockchain.scripthash.listunspent`
names token fields differently per server:

| Server | Token fields on `listunspent` |
|---|---|
| `cashnode.bch.ninja` (Fulcrum) | **none** |
| `rostrum.cauldron.quest` (Rostrum) | `has_token`, `token_id`, `token_amount`, `token_bitfield` |

Every consumer in the codebase read `utxo.token_data.{amount,category}`. Against
a Rostrum response they all read `undefined`, so the wallet reported
`token_categories_ft: 0` while **holding 2 confirmed ROACH** — and the Fulcrum
node the wallet was configured to use could not have told it otherwise even with
correct code. `sumFtBalances`, `utxoToTokenPrefix`, and `selectTokenUtxos` were
all correct; they were being fed nothing.

Fixed by normalising at the network boundary (`normaliseTokenData` +
`listUnspent()`) rather than teaching 14 call sites about field-name variants,
and by preferring the token-aware node in the mainnet server list.

**Rule:** a function that forwards a field to a dependency is not evidence the
dependency ignores it — sign one transaction and compare the bytes against a
transaction the network accepted. And a *discovery* bug is invisible to any test
that only exercises the thing you are trying to do: the send path was fine; the
wallet simply could not find its own tokens. A wallet that reports zero for a
balance you can see on-chain is the tell.

## 24. An implicit fee that fails in two opposite directions

`signP2pkhTransaction` computes the fee as `inputs − outputs`, so the change
amount *is* the fee budget. Nothing validates it, and it fails silently in both
directions:

- **Underpay.** 200 sat for a 242-byte transaction is 0.83 sat/byte. The node's
  `estimatefee` and `relayfee` both report 1e-05 (1 sat/byte), and the node
  rejects the whole transaction with `min relay fee not met (code 66)`. No txid,
  no partial effect, nothing spent.
- **Overpay into a negative fee.** Computing change as `input − fee` ignores
  that `createTokenOutput` silently raises the token output to the dust
  threshold (1000 sat here). The outputs then exceed the input and the "fee" is
  **negative**: `fee: -678`.

Neither failure names its cause. The node reports a fee problem or a token
problem depending on which mistake you made.

The same bug bit a second, larger time inside `send-token.mjs`. The token
outputs each claim sats (the dust floor again, 1000 sat each) **and** the FT
inputs carry sats of their own, but the change calculation was
`bchTotal - estFee`, counting neither. A 0.10 ROACH send produced outputs
exceeding inputs by **456 sat** — a negative fee, rejected with no clue. Sending
0.10 required knowing that two separate sats-claiming outputs existed, one of
them off-screen in a different code path.

**Rule:** assert the fee is positive and above the node's minimum relay fee
*before* signing, and compute it from the real output values rather than
assuming what a helper wrote. An implicit invariant with no assertion is a
latent loss, and a dust adjustment in a helper is exactly the kind of thing an
arithmetic chain forgets. Whenever a helper can silently *raise* a value —
`createTokenOutput` bumping to the dust threshold — read back what it actually
wrote rather than what you asked for.

---

---

## 25. A verification gate that checked direction but not magnitude

`verifyTransactionOutputs` byte-compared every output against the wallet's own
addresses. That correctly catches **redirection** — an output paying someone
else. It never compared the **value** of an output it had already proven ours.

A router quoting 36,141 and building an output paying our own address **1 base
unit** passes every existing gate: destination genuinely ours, pool count matches,
no foreign token category. The wallet signs it and receives nothing.

> Ownership answers *"is my money going somewhere I did not agree to?"* It cannot
> answer *"is my money arriving short?"* Different questions; only the second
> needs the amount.

Fixed with `expectedReceiveAmount` + `minReceiveAmount`, checked against the
quote **and** the user's `--min-output` floor, so the enforced bound is the one
the user consented to.

**And the amount is not always the sat value.** A CashToken output pays
`valueSatoshis: 1000` — the dust floor — while carrying 182 PUSD in its prefix. My
first fix compared sat values and rejected every *correct* token swap. The check
must read `token.amount` for token outputs.

**Rule:** a gate that proves *where* must not be described as proving *how much*.
Enumerate what your check does not establish, and ask which of those is worth
money to the counterparty you are trusting.

---

## 26. A stale-pool check that had never once run

Three bugs stacked, all in the same pre-signing check, all silent:

1. `request('blockchain.transaction.get', [txid, false])` — the client's `request`
   is **variadic**, so the array nested, the node answered `{}`, and `{}` is not an
   error string. Every pool input read "parent unknown" and the check became a
   no-op.
2. A **hand-rolled transaction walk** drifted on a 10,851-byte 57-in/56-out
   parent, returning the *same wrong locking script* for vout 4, 5 and 32 — the
   signature of a misaligned parse. It failed silently: the wrong lock still
   hashes to a valid scripthash, and a node asked about a script it does not
   index answers honestly with an empty set.
3. **A single node's empty answer was read as proof of spend.** Fulcrum does not
   index p2sh32 covenants: across 13 live pools, Rostrum saw 1–60 unspent each
   and Fulcrum saw none — 13 of 13 disagreed. Treating that as "spent" would
   refuse a valid swap.

Result: a swap passed every gate, was signed, and was **rejected at broadcast**
with `Missing inputs` because the pools went stale inside the run. Correct
response is to re-quote and rebuild, never to resubmit identical bytes.

**Rule:** a check that has never fired is not a check. Prove it fires by making it
fail on purpose — reintroduce the bug, confirm the test goes red, then restore.
And **verify your parser against a reference implementation before trusting it**;
`decodeTransactionBCH` was already imported one line above the hand-rolled walk.

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
| protocol `1.5` | a modern-looking version |
| `{}` from a call | a result |
| `request(m, [a, b])` | a params array |
| one txid byte order | the txid |
| receiving-only scan | the funded set |
| signing before checking | signing |
| a 69-byte scriptSig | an unsigned input |
| `node script.mjs` | running the tool |
| token fields named per server | a token balance |
| change = input − fee | a fee calculation |
| `BigInt("0.10")` on a display amount | a token send |
| `request(m, [a, b])` on a variadic client | a safety check |
| a fixed 20-address gap limit | a balance |

The defence is the same in every case: **push the rule down the mechanism
hierarchy** — type system → lint that fails CI → banned API → runtime check →
prose. A value-moving wallet should get its rules from the first four, and treat
prose as documentation of what the mechanisms already enforce. Most of the bugs
on this page existed precisely because a rule was living only in prose, or in a
comment that had drifted.
