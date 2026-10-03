---
pageType: reference
id: reference.security.dex-swap-integration
description: Security properties of integrating a BCH wallet with a CashTokens DEX — the operator-signing problem, the router's 10bps fee output, the assembler safety gate, slippage semantics of a CPMM DAG, and what an integrating wallet must verify locally before signing.
sourceUrl: https://docs.riftenlabs.com/router/api/tx.build/
---

# DEX and swap integration security on BCH

*How a wallet stays safe when it signs a transaction somebody else assembled. Written from the operator's side: what the Cauldron protocol structurally requires, what the Riften Labs Router discloses, what it refuses to build, and what the integrating wallet still has to verify itself. Complements [`cashtokens.md`](cashtokens.md) (token primitives) and [`wallet-threat-model.md`](wallet-threat-model.md) (key custody). Verified against docs.riftenlabs.com and the bch-bot `lib/router.mjs`, 2026-10-01.*

## 1. The operator-signing problem

A swap is not a wallet operation. A BCH wallet can build and sign an ordinary P2PKH send entirely on its own, but a Cauldron swap spends a **pool UTXO** as one of its inputs, and that UTXO is committed to the **pool operator's** public key hash. No user wallet holds that key. The wallet therefore cannot produce a valid swap alone — it must be handed a transaction, sign the subset of inputs it owns, and return it.

This is a property of the protocol, not a wallet limitation, and it inverts the usual trust model. The user is no longer the sole author of what they sign: a third party composes the inputs, the outputs, the fee outputs, and the ordering, and the wallet's signature is the last and only check. The security question for any integrating wallet is therefore **not** "is my key safe" but **"what exactly am I attesting to, and do I check it independently?"**

Riften Labs runs the **Cauldron Router** as the answer to this: a public, no-auth WebSocket service that assembles the unsigned transaction and states which inputs the client owns. The protocol is Electrum-style JSON-RPC over a single socket at `GET /v1/route`; there is no REST path for quotes or builds. Limits: max 1024 concurrent connections, 1 MiB max inbound message, 30 s per-frame send timeout, HTTP 503 on excess connections.

## 2. What the router discloses — and what it refuses to build

`tx.build` re-routes over the current snapshot (never a stale quote), assembles, and returns:

| Field | Meaning |
|---|---|
| `unsigned_tx_hex` | The unsigned transaction to sign |
| `source_outputs` | Prevouts, index-aligned to **all** inputs, for whole-tx signing |
| `inputs_to_sign` | Indices of the funding inputs the client actually owns |
| `expected_output` | Expected bought amount — the slippage reference |
| `fee_sats` / `fee_token_amount` | Router fee, one or the other non-zero, `"0"` where neither applies |
| `miner_fee_sats` | Estimated miner fee |
| `route` | The route the build was assembled for |

Two disclosure properties are genuinely good and should be credited. First, the router names the inputs you own **and** supplies the prevouts for all of them, so the wallet can sign the whole transaction under `SIGHASH_ALL` and let consensus bind every other input and output — you are not signing a partial SIGHASH that leaves the rest malleable. Second, the router **never broadcasts**. It returns an unsigned transaction and leaves broadcasting to the client, which means the wallet is the last checkpoint before the transaction is public.

**The assembler has a safety gate, and it fails closed.** Before returning anything it checks:

1. **No token burn** — every token category is conserved, `Σ in == Σ out`.
2. **Sane miner fee** — between the relay floor and a max-overpay band.
3. **Fee output present** — when a fee was charged, the output must exist at the fee address.
4. **No unspendable outputs** — all outputs have non-empty scripts and meet dust thresholds.

Any failure returns `build_failed` rather than an unsafe transaction. This is a real server-side defence: a client that only checks `inputs_to_sign` would still be protected from burning its own token balance by a buggy or hostile assembler. It is **not** a substitute for client-side checks — the gate protects against the assembler's mistakes and malice, but the wallet is the party that cannot afford the mistake.

**Fee disclosure.** Each built transaction carries a router fee as its own output, currently **10 basis points (0.1%)**, described as the on-chain fee for the trade itself and separate from service terms. The docs are explicit that the rate may change during beta and instruct clients to *"read what the build actually pays rather than hard-coding it."* This is the correct instruction and it is worth taking seriously: a hard-coded `100` sat fee assumption is a bug waiting for a rate change. Read `fee_sats` / `fee_token_amount` from the response every time.

Note the fee stack: **0.3% to LPs** (Cauldron's own trade fee) **plus 0.1% to the router**. The two are separate and both are real. See [`../entities/cauldron-dex.md`](../entities/cauldron-dex.md) for the LP-side math and note there that the fee literal in the Riften docs (`abs(satoshis of input - satoshis of outputs) * 0.03`) is a typo for `0.003`; trust the SDK's `calcTradeFee` or the on-chain opcodes, not the prose.

## 3. Slippage on a CPMM has different semantics than on EVM

Cauldron trades follow a **directed acyclic graph**, not a global state: every trade links directly to the previous one, and each pool has its own local state updated only by transactions that interact with it. Three consequences follow directly, and they are the honest reason a BCH DEX is less front-runnable than an EVM one:

1. **You know the exact price you pay, including slippage**, at build time. It is not affected by other interactions against a shared global state.
2. **There is no front-running in the EVM sense**, because there is no shared state for a miner to reorder against.
3. **If the transaction is accepted by a miner, the trade happens; if it is not, it is dropped and no network fee is paid.**

The downside is the mirror image of the last point: a miner may drop a transaction or select a conflicting one, and **every transaction chained on top of a dropped one is dropped too**. Cauldron's own docs call this out, and it is the reason "no RBF on a DEX trade" is not a free simplification. Conflicting trades can be submitted simultaneously by anyone; the miner takes the one they see first, the loser is dropped, and to the user it looks like a reversal — the previous asset simply comes back. Liquidity withdrawals, deposits and transfers chained after a dropped trade are lost the same way.

**Slippage floor semantics differ by side.** `min_output` is the only slippage control the protocol exposes, and it applies to `side: "sell"` only — in buy mode the output *is* the amount requested, so there is nothing to protect. An integration that exposes a slippage field on a buy-side order is showing a control that does nothing.

**A CLI whose amount is always denominated in the sell asset should use `side: "sell"`, and that is not a bug.** It is tempting to read a hardcoded `side: "sell"` as a buy/sell mix-up. It is not: the user names the asset they are spending and how much of it, so sell-side quoting is the correct semantic for the whole command surface. Buy-mode `min_output` is a control that does nothing, so hardcoding the sell side is what keeps the CLI honest. The guard belongs in one place — a boundary that throws when a floor is passed with a side that cannot enforce it.

### Live measurement: `market_pre_price` and `output_amount` disagree

Observed 2026-10-01 against the live router, 1 BCH → PUSD (2 decimals):

| Field | Value |
|---|---|
| `input_amount` | `99999735` (0.99999735 BCH) |
| `output_amount` | `36141` (361.41 PUSD) |
| `market_pre_price` | `3434.87` |
| `pools` | `68` |

`output_amount / input_amount` is **361.41**, but `market_pre_price` says **3434.87** — roughly 9.5x apart. The output is the reliable number: the implied rate is stable across input sizes (0.1, 1 and 2 BCH all returned ~360-362), and a reverse quote agrees on the same order of magnitude.

An integrator relaying both fields will surface the mismatch, because the two are in **different units** — the pool's marginal price in one scale and the trade's realised rate in another. Trust `expected_output`; treat `market_pre_price` and `market_post_price` as advisory only, and never render them to a user as the price they are paying until the units are reconciled with Riften. Note that `market_pre_price` is the field most likely to be used to show a user "you save X%" — that number is currently untrustworthy.

### Live measurement: broadcast host TLS is broken

`https://broadcast.cauldron.quest/broadcast` accepts a TCP connection on 443 and then fails the TLS handshake: `openssl s_client` reports `no peer certificate available`, and curl reports `tlsv1 alert internal error` in ~0.1s. The A record resolves and the router host on the same Cloudflare range serves normally, so this is the origin's TLS configuration, not a network path problem.

The practical consequence for an integrator: **quoting works while that particular broadcast path does not**, and the two fail independently. A wallet that tests its quote path against live infrastructure can look healthy while being unable to move funds. Test the broadcast path separately, and treat "quote succeeded" as evidence of nothing about broadcast reachability.

**Correction (2026-10-01, after further analysis).** This page originally framed the broken host as an *upstream blocker* on swaps, which was wrong in the only sense that mattered: it framed a single convenience endpoint as if it were the only way to broadcast. It is not. `blockchain.transaction.broadcast` over Electrum works on every node, including for CashTokens, so swapping is **not** blocked on the router's host recovering. The lesson generalises past this host: when an integration stops working, check whether the failing component is actually *required* or merely one path to it, before reporting the dependency as unavailable. See §4 "Broadcasting" for the measurements.

## 4. What an integrating wallet must verify locally

Everything below is the client's job, and none of it is delegated to the router. This is the checklist that a swap implementation should not be considered complete without.

**Before signing:**

- **Decoded inputs.** Walk every input in `unsigned_tx_hex`; the ones matching your funding UTXOs must be exactly the set you intended to spend. No inputs of yours may have appeared that you did not name.
- **Outputs.** Every output must be yours (recipient, change) or an expected third party (pool recreation at the same index as its input, fee output). An output to an address you cannot explain is the whole attack.
- **Amounts.** The recipient output equals the amount you asked for; the change output is the remainder after fees; nothing drained to an address you have never seen.
- **Token conservation, checked locally.** Every CashTokens category: `Σ in == Σ out`. Do not rely on the assembler's `build_failed` gate for this — that gate is the assembler grading its own work.
- **Fee outputs.** Every extra output, its value, and its recipient. Compare against what the build reported as charged.
- **Slippage floor.** If you set `min_output`, confirm `expected_output >= min_output` yourself before signing.
- **Quote-vs-build.** If you quoted first, confirm the built `expected_output` still matches. The router re-routes on every build, so a quote is a *reference*, not a commitment.
- **Recipient output *amount*, not just address.** Confirm the output paying your receive address carries the amount the build reported. The bch-bot audit of 2026-10-02 found this item was **not implemented**: `verifyTransactionOutputs` byte-compares locking scripts (which correctly catches redirection) but never compares the *value* of an output that is already ours, so a router that promises 36,141 and pays 1 base unit passes every gate. Verified by running the project's own gate against hand-built transactions. See [`pre-signing-invariant.md`](pre-signing-invariant.md) §2 for the reproduction and the fix shape.
- **Non-empty, well-formed input set.** A build that names zero inputs for you to sign is not a swap you should sign.

**Signing itself:**

- Sign with `SIGHASH_ALL | SIGHASH_FORKID` (0x41) for plain BCH inputs; add `SIGHASH_UTXOS` (0x20 → 0x61) for any token-bearing input. A signature that is well-formed but carries the wrong flag set is rejected by the network with `mandatory-script-verify-flag-failed` — the same misleading error you get for an unconfirmed parent. See [`script-and-signing.md`](script-and-signing.md) and, for the trap where a structurally valid signature is still rejected, [`../references/bch-signing-and-verification.md`](../references/bch-signing-and-verification.md): **verify the signature cryptographically, don't trust that encoding succeeded.**
- Prefer whole-transaction signing with the supplied `source_outputs` so every other input and output is committed to by your signature.

**Broadcasting:**

- **Broadcast to a wide endpoint, not a single node.** The router's docs are pointed about this: *"trades against the same pools chain on one another, so a transaction that reaches only part of the network is how double-spend conflicts start."*
- **You do not need `broadcast.cauldron.quest` at all.** It is a convenience HTTP endpoint, and it was returning **zero bytes over TLS** for an extended period: TCP connects on 443, then the handshake fails with `tlsv1 alert internal error` in ~0.14s. It is not part of the protocol and nothing depends on it.
- **Use Electrum instead:** `blockchain.transaction.broadcast` with the raw tx hex. It is part of the Electrum protocol, every full node serves it, and it needs no third-party HTTP service. Most wallets already hold such a connection open.
- **CashTokens-aware nodes broadcast token transactions correctly.** Verified directly rather than assumed: a transaction with a PUSD CashToken output broadcast to `cashnode.bch.ninja` (mainnet) and `chipnet.bch.ninja` returns `"rejected by network rules. dust (code 64)"` at 546 sats and `"rejected by network rules. Missing inputs"` at 2000 sats. Both are consensus-level rejections from a node that parsed the token prefix and reached UTXO lookup; a node that could not decode CashTokens would fail earlier with a decode error.
- **Do not infer token-awareness from `listunspent`.** The same servers return no `token_data` field for `blockchain.scripthash.listunspent` while accepting token-bearing broadcasts without complaint. UTXO enumeration and broadcast validation are separate code paths. Test the path you actually depend on.
- If you keep the HTTP endpoint as a fallback, treat **both** failures as a broadcast failure and report both — and note that a broadcast failure is *not* a signing failure. The signed transaction is valid and can be relayed by hand, which is the most useful thing you can tell a user at that moment.
- Do not treat a successful broadcast as a settled trade. Until it is mined, it is pending, and the DAG semantics above apply in full.

### What a real multi-pool swap's outputs actually look like

This is the part that is easy to get wrong, and worth reading before writing an output verifier. Measured against the live router with a 1 BCH → PUSD build: **31 outputs** for a route through 28 pools.

| Outputs | Shape | What it is |
|---|---|---|
| 0–27 | 35 bytes, `aa 20 <32-byte hash> 87` | **Pool covenants.** p2sh32 locking bytecode: `OP_HASH256 OP_PUSHBYTES_32 <32> OP_EQUAL`. One per pool in the route. |
| 28 | 25 bytes, `76 a9 14 …` | P2PKH to **your** receive address. |
| 29 | 35 bytes, same shape as a covenant | **The router fee** — 998 sats in the measured build. Not a pool. |
| 30 | 25 bytes | P2PKH to **your** change address. |

Four consequences for anyone verifying outputs, each of which cost a real debugging cycle:

1. **Shape does not identify ownership.** A 35-byte covenant is *usually* a pool but is sometimes the fee, and the change output can be a covenant too. Ownership must be decided **only** by byte comparison against the addresses you supplied, and that verdict must win over any shape heuristic. A verifier that classifies by shape and then checks ownership will eventually classify one of your own outputs as foreign.

2. **Covenant output values are not satoshis.** The 29 covenant outputs in the measured build sum to **38,035,876,708** — three orders of magnitude more BCH than the transaction contains. Each value is that pool's **token position in base units**, surfaced by the decoder in the same field as an output's satoshi value. Summing them naively makes a check "detect" 38,000 BCH leaving a transaction that only ever held 0.05 BCH. **Exclude covenant values from any satoshi ceiling or conservation check.**

3. **Conservation is about what leaves to addresses you do *not* control.** In the measured build *both* P2PKH outputs are yours — 1,000 sats of PUSD and 3,994,173 sats of BCH change. "Inputs minus all P2PKH outputs" is therefore *not* the fee; it also subtracts your own change, which made the implied fee look like 1,004,827 sats instead of the real 6,673. Count only non-ours P2PKH outputs as leakage.

4. **The pool-count check must be one-directional.** More covenant outputs than the quoted pool count is a problem (value is being committed to liquidity the user did not agree to). Fewer is fine — a router may net several pools' inputs into one output. Allow for the fee output, which shares the covenant shape.

The general rule: **a verifier written before anyone has decoded a real transaction will be wrong about the transaction it verifies.** The fix is to read live output and pin that shape as a fixture, not to reason about what the protocol "should" produce.

## 5. Operator-specific risk: the DEFi derivation chain

Cauldron derives pool contract addresses from a **non-standard HD chain at index 7**:

```
m/44'/145'/0'/0/<i>   receive   (BIP-44 standard)
m/44'/145'/0'/1/<i>   change    (BIP-44 standard)
m/44'/145'/0'/7/<i>   DeFi      (non-standard, Cauldron liquidity)
cauldron_address = getCauldronContractAddress(hash160(pubkey at /7/i))
```

The security implication for any tool that scans or recovers wallets: **a standard wallet only scans `/0/` and `/1/`.** A pool contract address derived from `/7/` is not a normal P2PKH output and will not be discovered by ordinary scanning. Cauldron mitigates by scanning up to 50 addresses ahead of the last used index per chain, so gaps are usually found automatically — "usually" being the operative word. Any recovery tooling that assumes only BIP-44 chains exist will silently under-report what a seed controls.

The pool address is deterministically tied to the mnemonic like any other address, which is the property that makes an LP recoverable after a crash. It also means the `/7/` chain is **not a separate secret** — anyone with the 12-word phrase controls every liquidity position, which is exactly why Cauldron's docs attach a "keep your mnemonic safe" warning to the HD-wallet page.

## 6. Auditability: what an integrator cannot check

- **The assembler is unaudited.** The safety gate is documented, which is more than most builders offer, but no third-party review of the Router or the pool contracts was found. "Documented to have a gate" is not "verified to have a gate".
- **Fee rate is explicitly unstable.** "Currently 10 bps… may change during beta" means historical on-chain fee outputs are the only reliable record. Read every build; never infer the fee from the docs.
- **The 0.3% prose is wrong** in the official docs (`0.03` for `0.003`). If your integration quotes fees from documentation text rather than from `calcTradeFee` or the returned `fee_*` fields, it is wrong today.
- **Pool state comes from an indexer.** Discovering which pools exist and what they hold is a Rostrum/indexer query. An indexer that lies about pool contents produces a quote that cannot fill — the transaction fails rather than silently mispricing, which is the safe failure mode, but it means pool availability is only as trustworthy as the indexer you queried.

## What's not solved

> **Addendum, 2026-10-02 (live-code audit).** Two items in the "auditability" list
> above are now concrete rather than theoretical, and one is worse than "cannot be
> checked". A hostile router can pass every local check by **under-delivering
> rather than redirecting** — reporting the right `expected_output`, satisfying
> `min_output` in the reported figure, and paying a near-zero amount to your real
> receive address. Ownership is verified; **amount is not**. Separately, a `{}`
> response to `blockchain.transaction.broadcast` is still logged as
> `"broadcast": true` in five of the bot's scripts — the same non-answer the
> `failure-modes-we-hit.md` §16 catalogue records, fixed in `lib/router.mjs` and
> never applied to its siblings. Both are documented in
> [`pre-signing-invariant.md`](pre-signing-invariant.md) and
> [`key-custody-and-oracle.md`](key-custody-and-oracle.md).

- **No atomic swap.** A trade is a single transaction whose validity is enforced by the covenant. There is no hash-time-locked fallback, so the only recovery from a dropped trade is that the pool UTXO is unspent and your inputs are yours again — which is a property of the DAG model, not a refund mechanism with a deadline.
- **Miner censorship is unresolved.** "No front-running" is a structural property. "A miner will include your transaction" is not, and the dropped-cascade is explicitly documented as a real outcome.
- **Slippage protection is buy-side blind.** No floor on the output of a buy, by construction.
- **No operator key separation.** The same `/7/` chain produces every pool address for a given seed. Compromise of the seed compromises every position simultaneously; there is no per-pool key to isolate.
- **Fee and pool data are unverified third-party inputs.** Both are disclosed by the operator and neither is independently checkable before signing — only the *outputs* of a build are checkable, and only after it exists.

## Sources

- Cauldron Router `tx.build`: <https://docs.riftenlabs.com/router/api/tx.build/> — safety gate, fee output, signing aids, error codes
- Cauldron Router protocol overview: <https://docs.riftenlabs.com/router/protocol/overview/> — JSON-RPC transport, limits, amount and asset conventions
- Cauldron Router error codes: <https://docs.riftenlabs.com/router/protocol/errors/> — `build_failed`, `slippage`, `route_failed`, subscription errors
- Cauldron trade/swap mechanics: <https://docs.riftenlabs.com/cauldron/swap/> — k-invariant, 0.3% LP fee, multi-pool trades
- Cauldron trade ordering (DAG): <https://docs.riftenlabs.com/cauldron/knowledge-base/txordering/> — local state, no front-running, EVM comparison
- Cauldron pending transactions: <https://docs.riftenlabs.com/cauldron/knowledge-base/pending/> — conflicting trades, reversal semantics, dropped cascade
- Cauldron HD wallet & derivation paths: <https://docs.riftenlabs.com/cauldron/knowledge-base/hdwallet/> — `/7/` DeFi chain, gap-limit scanning
- Broadcast API: <https://broadcast.cauldron.quest/broadcast>
- Cauldron fee arithmetic on-chain: `../sources/cashlab-cauldron.md` (`packages/cauldron/src/util.ts`, `calcTradeFee`)
- bch-bot router client: `lib/router.mjs` (`verifyBuildAgainstQuote`, `min_output` side check, broadcast error handling)


---

## Addendum: what a live swap actually did (2026-10-02)

Everything above is design review. This is what happened when a real swap was
built and broadcast, and it corrects two assumptions.

### The gate verified *where* value went, never *how much*

`verifyTransactionOutputs` byte-compares every output against our addresses, which
correctly catches **redirection**. It did not compare the **value** of an output
already proven ours. A router that quotes 36,141 and builds an output paying our
own address **1 base unit** passes every existing gate: the destination is
genuinely ours, the pool count matches, no token category is wrong. The user signs
it and receives nothing.

> Ownership answers *"is my money going somewhere I did not agree to?"* It cannot
> answer *"is my money arriving short?"* Those are different questions, and only
> the second one needs the amount.

Fixed with `expectedReceiveAmount` + `minReceiveAmount`, enforced against the
amount the **quote** promised and the floor the **user** set via `--min-output`.
The bound enforced is therefore the one the user actually consented to.

**A CashToken output's amount is not its sat value.** A PUSD output pays
`valueSatoshis: 1000` — the dust floor every token output is raised to — while
carrying 182 PUSD in its `0xef` token prefix. Comparing sat values against the
quote rejected every *correct* token swap. The check must read `token.amount` for
token outputs and `valueSatoshis` only for plain BCH. Verified against a live
BCH → PUSD build: `valueSatoshis 1000n`, `token.amount 182n`, quote said 182.

### The stale-pool check had never run, and was silently disabled

Three separate bugs, found in sequence, all in the pre-signing pool check:

1. **`request(method, [candidate, false])`** — `@electrum-cash/network`'s `request`
   is *variadic*. Passing an array nests it, the node answers `{}`, and
   `{}` is not an error string — so all 13 pool inputs read "parent unknown" and
   the check degraded to a no-op. `request(m, a, b)`, never `request(m, [a, b])`.
2. **A hand-rolled transaction walk.** The pool parent is 10,851 bytes with 57
   inputs and 56 outputs; hand-computed byte offsets drifted and returned the
   **same wrong locking script for vout 4, 5 and 32**, having slipped inside a
   CashToken prefix. Identical wrong bytes for different indices is the signature
   of a misaligned parse. It failed *silently*: the wrong lock still hashes to a
   valid scripthash, and a node asked about a script it does not index answers
   honestly with an empty set. All 13 live pools came back "already spent". Use
   `decodeTransactionBCH`, already imported for the unsigned transaction.
3. **A single node's empty answer was read as proof of spend** — see executive
   point 14. Now `scriptHasUnspent()` returns `unspent | spent | inconclusive`,
   and `spent` requires **two independent nodes** to agree.

After all three, a live quote reported `pool inputs verified unspent (13/13)` —
the first time the check had ever actually run.

### Sign the chain the address belongs to

`swap.mjs` derived input keys with `change=0` unconditionally, so a change-chain
UTXO was signed with the **receiving**-chain key at the same index (`/0/19` not
`/1/19` — verified different keys and different addresses). The signature fails to
validate and the rejection reads like a malformed transaction, not a wrong key.

This stayed hidden by coincidence, not design: the funding scan reached change
index 19 while `change_index` was 40, so every change UTXO it could see happened
to sit on the receiving chain. That stops being true the moment the gap closes.
`lib/wallet.mjs` already exports `resolveAddressPath`, which resolves the full
account/change/index path and **throws** when an address is in neither chain rather
than guessing.

### A swap also needs a change address, and it must not be wasted

A swap derives a change address at signing. Reserving it there means a *rejected*
swap still consumes one. See
[`../references/resource-safety-and-wall-clock.md`](../references/resource-safety-and-wall-clock.md)
— one rejected swap advanced `change_index` by 5. The address must be committed
only after the node returns a real 64-hex txid.


---

## Correction: `Missing inputs` was never attributed to a cause

The addendum above says the live swap "was rejected at broadcast because the
pools went stale inside the run." **That cause was assumed, not established**, and
it should not have been written as fact here.

What the log actually contained: three attempts, each returning a **byte-identical
quote — the same price to 27 decimal places**. Contended pools move the price. That
is not a subtle inference; it is the refutation, sitting in the same log I read
three times.

Decoding the signed transaction settles what is *not* the cause:

    inputs: 15   outputs: 15
    scriptSig per input: [69 × 13, 100, 100]     → 15/15 signed
    ours (13, 14):  100 bytes, sighash 0x41
    pool (0–12):     69 bytes, sighash 0x7c     → pre-signed by the operators

The transaction is fully signed. The 13 pool inputs arrive already signed, and
their prevouts are the router's responsibility, not ours. So a `Missing inputs`
here points at the *prevouts the router selected* — not at our keys, not at our
signing, and not demonstrably at contention.

Still unresolved: whether those pool UTXOs are genuinely spent, or whether the
broadcasting node cannot evaluate p2sh32 covenants. The first is a router/chain
fact; the second is a node-capability fact. They have different fixes, and I never
distinguished them.

**A `Missing inputs` rejection names a symptom, not a cause.** ABC reports one
error for an input that is unknown and for an input that is already spent. Before
writing a cause into a design document, require a fact that could only be true
under that cause. For contention that fact is price movement across attempts.


---

## Structural finding: 13 pools is two transactions, not thirteen

Decoding the signed swap's outpoints:

    381179a50a23…02fd  →  v1, v3, v4, v5, v7, v10, v11, v12, v14, v15, v32, v35
    5a34d4d75679…7686  →  v0

A quote spanning 13 pools consumes outputs from **two** parent transactions. A
swap is not 13 independent settlements that can each go stale independently —
there are only two spendable parents, and a single conflicting spend of either one
invalidates the whole route. That reframes contention risk: it is not "any of 13
pools may be taken", it is "two specific transactions must be unspent", and
checking them is two queries rather than thirteen.

This is the one durable result of the 2026-10-02 swap investigation, and it came
from decoding the transaction rather than from reading error messages.

**What the investigation did not establish:** the cause of the `Missing inputs`
rejection. My probes for it were themselves broken — the same queries that
reported pool parents as nonexistent also reported a *known-confirmed* wallet
txid as nonexistent. See
[`../syntheses/failure-modes-we-hit.md`](../syntheses/failure-modes-we-hit.md)
§28 for the three client bugs and why a control is mandatory. Treat any earlier
claim about the rejection's cause as unverified.
