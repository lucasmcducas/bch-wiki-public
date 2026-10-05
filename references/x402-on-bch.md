---
pageType: reference
id: reference.x402-on-bch
description: x402 (HTTP 402 micropayments) assessed for Bitcoin Cash — the protocol's V2 mechanics, why BCH is absent from it, the two mutually incompatible BCH dialects that exist outside it, the 546-sat dust floor that decides the whole design, and what this actually means for the Omarchy wallet. Written 2026-10-05 from primary sources plus live probes.
sourceUrl: internal/reference
---

# x402 on BCH

x402 is the open standard that turns the long-dormant HTTP `402 Payment Required`
status into a machine-usable payment layer, so an autonomous agent can pay for an
API call without an account, a card, or an OAuth dance. It is large, real, and
institutionalised: 40 Linux Foundation members including Stripe, Visa, Google,
Cloudflare and AWS.

It also has no Bitcoin Cash support. That is the accurate summary, but it is the
*least* interesting part of this page, because the interesting part is what
happened at the edges — two incompatible BCH dialects, both built outside upstream,
one of them running in production, and an economics result that forces a specific
design on anyone who tries this.

Everything below was checked against primary sources and live endpoints on
2026-10-05. Where a claim could not be reproduced, it says so.

## Read in this order

1. **The protocol, briefly** — enough to reason about a new rail.
2. **BCH is not in it** — and is not merely omitted from a list.
3. **The two dialects that exist anyway** — and why they cannot talk to each other.
4. **The 546-sat floor** — the number that decides everything downstream.
5. **What this means for the wallet** — including the feature we should refuse.

## 1. The protocol

x402 V2 communicates **entirely in HTTP headers**, all base64-encoded JSON:

| Header | Direction | Carries |
|---|---|---|
| `PAYMENT-REQUIRED` | server → client | the `PaymentRequired` challenge |
| `PAYMENT-SIGNATURE` | client → server | the `PaymentPayload` |
| `PAYMENT-RESPONSE` | server → client | the `SettlementResponse` |
| `EXTENSION-RESPONSES` | facilitator → server | extension outcomes; never forwarded to buyers |

The flow: request → `402` + `PAYMENT-REQUIRED` → client builds and signs a payload
→ retry with `PAYMENT-SIGNATURE` → server verifies (locally or via a facilitator)
→ settles → `200` + `PAYMENT-RESPONSE`. `accepts[]` is an **array**, so a server
advertises every `(scheme, network)` it will take and the client picks.

**Cite V2, not V1.** The `X-PAYMENT` / `X-PAYMENT-RESPONSE` headers in almost every
tutorial, including Coinbase's own launch material, describe V1. V2 also moved
network IDs to CAIP-2 (`eip155:84532`, not `base-sepolia`) and restructured the
payload so requirements nest under an `accepted` object. Facilitators accept both
during migration. Spec versions: v0.1 2025-08-29, v0.2 2025-10-03, **v2.0
2025-12-09**.

A **facilitator** verifies and settles on the server's behalf so servers need no
chain connectivity. It is explicitly *not* a custodian. Two methods matter:
`/verify` is read-only and MUST NOT write onchain; `/settle` commits. Anyone can
run one, and `GET /supported` publishes the facilitator's own settlement addresses
per network — which is how a client checks it is not being redirected.

Three payment schemes: `exact` (fixed price), `upto` (buyer authorises a maximum,
seller charges actual), and `batch-settlement` (off-chain cumulative vouchers,
batched onchain redemption — **EVM and Solana only**).

Governance: Coinbase launched it 2025-05-06, moved it to the **x402 Foundation**
under the Linux Foundation, which became operational 2026-07-14. Repo activity is
high (6.6k★, pushed the same day).

## 2. BCH is not in x402 — and it is not merely absent from a list

This is stronger than "not listed." Verified four ways on 2026-10-05:

- The official network list enumerates **twelve** implementations — EVM, Solana,
  TON, Algorand, Stellar, Aptos, Hedera, Keeta, NEAR, Concordium, XRPL, Cardano.
  No BCH.
- The scheme spec's own prose lists the network implementations, and its only
  Bitcoin-family entry is **`lnbtc` — Bitcoin Lightning**, `exact` over BOLT11
  preimages. There is **no `bch` CAIP-2 namespace anywhere**.
- The SDK mechanisms directory contains exactly `aptos avm cardano casper
  concordium evm hedera keeta near stellar svm tvm xrpl`. No `bch`.
- Neither x402 nor x402-rs contains a BCH mechanism package.

So: **no support, no namespace, no roadmap item, nothing to integrate against
upstream.** Treat any claim that "x402 supports BCH" as false unless it names a
non-upstream package.

The `lnbtc` detail is worth a second look, because it is the closest thing to a
precedent. Bitcoin Lightning *is* specified for `exact` — and it is
**spec-only**: no implementation exists in any SDK. So even the one Bitcoin-family
rail that made it into the spec has no code behind it. That is the realistic
ceiling for adding a UTXO chain to x402, and it is higher than people assume.

## 3. Two incompatible BCH dialects exist anyway

This is the part the upstream search does not find, because both were built
outside the project. They are **not wire-compatible** and a stock `@x402/core`
client cannot pay one using the other.

### Dialect A — `utxo` scheme, BIP-122 IDs, prepay-and-debit. In production.

An org of four repos ([spec](https://github.com/x402-bch/x402-bch),
[facilitator](https://github.com/x402-bch/x402-bch-facilitator),
[express](https://github.com/x402-bch/x402-bch-express),
[axios](https://github.com/x402-bch/x402-bch-axios)) — verified via the GitHub
API at 5 / 0 / 1 / 1 stars, last pushes between 2025-12-23 and 2026-08-03. Tiny,
and consistently so.

Its design is the important part. Rather than paying per request onchain, a client
**prefunds a UTXO** and the facilitator decrements an off-chain ledger across many
requests. Network IDs are BIP-122 genesis-hash form
(`bip122:000000000000000000651ef99cb9fcbe`), scheme `utxo`, CashAddr `payTo`.

Its production user is **Paytaca**, whose `paytaca-cli` (TypeScript, npm
`paytaca-cli`, 97 `x402` references in the shipped dist, last modified
2026-09-22) implements this and ships agent skills. Paytaca markets metered
LLM inference priced in satoshis with "buy time, not tokens" accounting — and
charges a platform fee as a second output with a 546-sat floor, which is direct
evidence they hit the dust limit in production.

> **Verification note — read this before citing the endpoint.** The Paytaca API
> host is live: `https://api.paytaca.ai/v1/models` returns `200` from an
> independent host. But **no x402-gated route on that host returned `402` for me.**
> `/api/quote`, `/api/ai/purchase`, `/api/ai/lift-quote` and
> `/v1/chat/completions` all returned `404`; `/api/quote` and `/api/generate` are
> the routes its own bundled test server implements, and that server is meant to
> run locally. So a genuine `402` with `"scheme":"utxo"` and a CashAddr `payTo` has
> been observed from this Paytaca deployment and is quoted in the project's own
> release notes, **but I could not reproduce it unauthenticated on 2026-10-05.**
> Treat "a production BCH x402 service exists" as well-supported and "you can go
> hit it right now" as **not verified**. The route may be auth-gated, moved, or
> retired.

### Dialect B — `exact` scheme, `bch:` IDs, full signed transaction. Open PR, not merged.

[`@optnlabs/x402-bch`](https://github.com/OPTNLabs/x402-bch) — Apache-2.0,
published to npm 2026-10-01, **2 stars**, single contributor, plus a fork. Backs
an **open, unreviewed** upstream PR ([x402#3632](https://github.com/x402-foundation/x402/pull/3632),
1 commit, +3359 lines, 0 reviews) and an issue (#3633) asking maintainers how to
review it. An earlier PR (#3609) and an x402-rs PR (#128) were closed unmerged as
superseded.

The code is genuinely good: a reviewer ran the suite and got **161 tests passing**
across 7 files, including 117 Rust-parity vectors, with a clean `tsc` and build.
It pins `@bitauth/libauth` at exactly `3.1.0-next.8` — **the same version and
same prerelease `bch-bot` uses**, so there is no libauth split to resolve. It
targets `@x402/core ^2.28.0`.

But two things deserve scepticism, and both are checkable:

- **Its spec contradicts its README.** Both advertise "native BCH, fungible
  CashTokens, NFT capabilities/commitments, P2SH20/P2SH32." The normative spec in
  the same PR says the opposite in its own words: *"CashTokens, CashScript, PSBT,
  sponsorship, non-P2PKH scripts, batch settlement, and alternate address
  encodings are outside this scheme"* and *"CashToken-bearing source outputs,
  inputs, or outputs MUST be rejected."* Types for token requests exist in code,
  so the plumbing is there — but **treat CashTokens and P2SH32 as claimed, not
  shipped.**
- **Its cited chipnet transactions could not be verified.** Five independent
  explorers returned anti-bot pages, SPAs or connection failures. Plausible, not
  confirmed.

There is also a third cluster — `bch-x402-gateway` (2 commits, design memo), a
`BCHx-api` "OpenAPI spec" that turns out not to use HTTP 402 at all, and a Rust
`cashr` (client-only, stale since April) — none of which is worth building on.

### Why the distinction matters

Dialect A settles **off-chain against a prefunded UTXO**. Dialect B ships a
**complete pre-signed transaction** per request. These are different payment
models, not different implementations of one. Picking the wrong one is a rewrite,
which is why the next section is not optional.

## 4. The 546-sat dust floor decides the design

This is the number that makes the whole question tractable, and it is not what
people expect. **Fees are not the blocker; the minimum relayable output is.**

Measured against a live BCH mainnet Fulcrum node, 2026-10-05:

- **Fees:** `blockchain.estimatefee(1000)` → `1e-05` BCH/kB = **1 sat/byte**. A
  1-in/2-output transaction (~148 vB) costs **148 sat**, about **$0.00047** at
  $316/BCH. Cheap.
- **Dust:** **546 sat**. BCHN's policy is `DUST_RELAY_TX_FEE = 1000 sat/kB`, and a
  546-byte P2PKH output divided by 1000 gives 546 sat — the smallest output the
  network will relay. At $316 that is **$0.0017**; at $640 it is $0.35.

So a single on-chain BCH payment cannot be smaller than the dust floor. **Sub-cent
x402 micropayments on BCH are not a pricing question, they are a network rule.**

The fee *ratio* is what bites. Recomputed from live figures:

| payment | $ value | fee overhead |
|---|---|---|
| 546 sat (dust) | $0.0017 | **27%** |
| 1 000 sat | $0.0032 | 14.8% |
| 5 000 sat | $0.0158 | 3.0% |
| 10 000 sat | $0.0316 | 1.5% |
| 15 000 sat | $0.0474 | **~1%** |

Overhead only falls under 1% past ~15 000 sat. Per-request settlement below that is
economically silly.

### Block time kills the synchronous round-trip

Measured over 11 consecutive mainnet headers: intervals of 40 s to 1 333 s, mean
**7.55 min**, median 406 s. The 10-minute target is incompatible with waiting for an
API response.

The `exact` spec is honest about this: a reachable-but-unaccepted transaction MUST
return `settlement_pending:<txid>` and MUST NOT be treated as paid — and a pending
response **cannot serve a synchronous API call**. Its default is one confirmation.
A `noDoubleSpendProof` mode exists, but the spec insists a double-spend proof is
"conflict evidence, not confirmation."

**Dialect A sidesteps this entirely**, and that is its real insight: it never waits
for the *new* payment to confirm. It confirms **once**, at onboarding, then debits
off-chain. Block time is paid once, not per request.

### There is no BCH Lightning

Searched specifically: **no BCH-native instant-finality rail exists.** Everything
found was Bitcoin Lightning, a different chain. Payment-channel work on BCH exists
only as scattered abandoned experiments (e.g. `fifikobayashi/Debt-Covenant-Contract`,
8★, last push 2020).

**Conclusion:** the only economically viable BCH x402 is **prepay-and-debit** — one
onchain payment amortised over N requests. This is not a workaround; it is the
design, and Dialect A already ships it in production.

## 5. What this means for the wallet

The wallet in question is `bch-bot` plus its Omarchy plugin: self-custodial, no
accounts, no KYC, no OAuth, all wallet logic in the CLI behind an enforced
QML/CLI boundary.

**First, the framing correction that kills most of the feature list: this wallet is
an x402 *client*, never a server.** Wallet users do not sell API access. "Metered
paid API for wallet users" is a seller-side feature and there is nothing here to
sell. Most "x402 for this product" ideas evaporate at that sentence.

**What Dialect B would fit, if anything.** Its payment object — a complete
client-signed UTXO transaction — maps onto a seam that already exists:
`lib/sign.mjs` exports `signExternalTransaction()`. The transport is Electrum, the
same JSON-RPC family as the package's `FulcrumProvider`. And because the payload is
a full signed transaction rather than a delegation, there is no bounded authority
to hand to a third party — the existing human-approval model survives intact.

**The feature everyone will ask for is the one we should refuse.** x402 exists for
autonomous machine payment. The wallet's own WizardConnect code says:

> There is deliberately no `--yes` flag. A wallet that can be told to sign without
> a human is not a wallet.

Every BCH x402 payment is a full broadcast, so agent-to-agent payments require a
pre-funded hot signer with programmatic signing and spend limits. That is exactly
the self-custodial property the product exists to keep. **Reject it explicitly**,
or someone will add it as a "small convenience."

Also reject: x402 paywalls on the plugin's own remote features (charging users for
infra the maintainer hosts, plus a server and key custody), paid plugin
distribution (the marketplace is GitHub and the plugin is MIT), and x402 for
Cauldron router calls (third-party and already free — adding a payment layer buys
nothing and adds a failure mode to a path already degraded by the
`broadcast.cauldron.quest` TLS failure).

### The honest verdict

**Technically possible: yes, and partly demonstrated. Worth building: not yet.**

Three reasons, in order:

1. **Nobody to talk to.** The open question is merchant-side. Until a BCH merchant
   publishes a route we can actually reach, an x402 client is a client with no
   server. Watch for that, not for the client.
2. **Two dialects, no standard.** Building on Dialect B means depending on a
   2-star, single-contributor, unmerged package whose spec contradicts its own
   README. Vendoring at a pinned SHA is defensible for a spike and indefensible as
   a shipped dependency. Building the scheme ourselves is weeks and a security
   surface we would own — and there is no evidence anyone would pay for it.
3. **The economics only work prepay-and-debit**, which is Dialect A's model, and
   Dialect A is the one already in production on the other side of a wall we could
   not get past.

If revisiting: the cheap first step is not writing the feature. It is standing up
Dialect B's facilitator on **chipnet**, running its own offline fixtures against
`bch-bot`'s signer to prove behavioural parity, and attempting one real 402 against
a route we can reproduce unauthenticated. If that cannot be made to work, the
question is settled without writing a line of integration code.

## Provenance and what is not verified

Verified from primary sources and live probes on 2026-10-05: the V2 headers and
CAIP-2 IDs; the twelve-network list and the `lnbtc`-only Bitcoin entry; the
mechanisms directory contents; the 546-sat floor and 1 sat/byte fee policy; BCH/USD
at $316.25; the `@optnlabs/x402-bch` package pin on `3.1.0-next.8`; PR #3632 open
with zero reviews; the four `x402-bch` org repos' stars and push dates; the live
`/v1/models` 200 and the 404s on every x402 route we tried.

Not verified: the Paytaca 402 response (real per release notes, not reproducible
unauthenticated by us); the three cited chipnet transactions (no working explorer);
CashTokens/P2SH32 support in Dialect B (spec forbids what the README advertises);
any funding figure for the x402 Foundation, which no primary source publishes; and
x402's homepage transaction/volume counters, which are self-reported with no
stated methodology.

Also note: `mempool.space`'s `minimumFee` is **Bitcoin's** API. The BCH figures
above come from a BCHN Fulcrum node.

## See also

- [[references/in-wallet-swaps]] — the Cauldron AMM path, and why a payment layer
  on top of an already-degraded broadcast path is not attractive
- [[references/receiving-an-address]] — the address transports on the receive side
- [[concepts/cash-tokens]] — what the CashTokens support in Dialect B would even mean
- [[references/wizardconnect/what-it-is]] — the human-approval model that agent
  payments would undermine
- [[security/wallet-threat-model]] — why a hot signer is a compromise, not a feature