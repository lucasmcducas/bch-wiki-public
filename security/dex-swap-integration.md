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

The practical consequence for an integrator: **quoting works while broadcasting does not**, and the two fail independently. A wallet that tests its quote path against live infrastructure can look healthy while being unable to move funds. Test the broadcast path separately, and treat "quote succeeded" as evidence of nothing about broadcast reachability.

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
- **Non-empty, well-formed input set.** A build that names zero inputs for you to sign is not a swap you should sign.

**Signing itself:**

- Sign with `SIGHASH_ALL | SIGHASH_FORKID` (0x41) for plain BCH inputs; add `SIGHASH_UTXOS` (0x20 → 0x61) for any token-bearing input. A signature that is well-formed but carries the wrong flag set is rejected by the network with `mandatory-script-verify-flag-failed` — the same misleading error you get for an unconfirmed parent. See [`script-and-signing.md`](script-and-signing.md) and, for the trap where a structurally valid signature is still rejected, [`../references/bch-signing-and-verification.md`](../references/bch-signing-and-verification.md): **verify the signature cryptographically, don't trust that encoding succeeded.**
- Prefer whole-transaction signing with the supplied `source_outputs` so every other input and output is committed to by your signature.

**Broadcasting:**

- Broadcast to a wide endpoint, not a single node. The router's docs are pointed about this: *"trades against the same pools chain on one another, so a transaction that reaches only part of the network is how double-spend conflicts start."* Use `POST https://broadcast.cauldron.quest/broadcast` with `{"tx": "<hex>"}` and expect a `txid` back; treat a non-2xx or unparseable response as a failure, not a success.
- Do not treat a successful broadcast as a settled trade. Until it is mined, it is pending, and the DAG semantics above apply in full.

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
