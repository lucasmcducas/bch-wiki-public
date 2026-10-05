---
pageType: reference
id: intents.zcashto-cash
description: zcashto.cash — the Zcash on/off-ramp built on Peer and NEAR 1Click, and the closest working precedent for a Bitcoin Cash equivalent. Covers the five-step cash-out, why the inventory model matters, what "shielded" does and does not protect, and the one thing it deliberately is not. Written 2026-10-05.
sourceUrl: https://zcashto.cash/llms-full.txt
---

# zcashto.cash: the precedent, working

A Zcash onramp and offramp that pays out to **Zelle, Revolut, Monzo, Alipay,
Chime and Mercado Pago** — and buys ZEC with Cash App, Zelle, PayPal, Wise,
Revolut and Monzo. It is the only consumer product found that completes the
flow this wiki cares about: **non-EVM privacy coin in, everyday payment app out.**

For a BCH wallet this is the most useful page in the section, because it is a
completed version of the thing being considered. Read it as a template.

## It is a thin front end, not a new protocol

> Peer-to-peer activity powered by Peer.

Everything underneath is already documented:
[ZKP2P attestation for the fiat leg](zkp2p-peer.md) and
[NEAR Intents for the non-EVM hop](near-intents-bch.md). zcashto.cash adds the
parts neither of those does: a one-time deposit address per request, a cash-out
form, resumable recovery, and the shielded delivery option.

That is the most transferable fact on this page. **Nobody built a new protocol to
do this.** They wrote a front end over two existing ones.

## The cash-out flow

From the product's own documentation:

| | |
|---|---|
| **01** | You get a **one-time ZEC address** |
| **02** | Your ZEC becomes **Base USDC** — after one Zcash confirmation, pre-positioned inventory delivers it |
| **03** | You create a **Peer cashout** — choose where to get paid |
| **04** | A Peer **taker pays you** with fiat and proves the transfer |
| **05** | You receive cash, or cancel and return the USDC |

Two details are doing real work.

**The one-time address is per request.** Not a deposit address you keep. That is
the same freshness property our receive view already uses for privacy, and it
means two cash-outs can never be linked by address reuse.

**"Pre-positioned inventory" is the important phrase.** zcashto.cash does **not**
route your ZEC per transaction. Inventory sits ready on Base, and **NEAR 1Click
replenishes it in the background.** Your ZEC is converted against stock that is
already there.

This is the same architectural dodge that makes the flow cheap, and it is the
single most important lesson here: **the 1Click route is amortised across many
users rather than paid per request.** A per-request NEAR quote at ~150 seconds
would be unusable. A background replenishment is invisible and nearly free.

## What "shielded" actually protects — and what it does not

Zcash shielding is the reason this product is interesting, and the honest
version is narrow.

**Protected:** funding from shielded inputs does not reveal the shielded sender
address in public Zcash data. Buy delivery supports Unified `u1` addresses, where
the transaction exposes neither recipient nor amount.

**Not protected**, per the product's own disclosure:

- The **cash-out funding path uses a one-time transparent Zcash address**.
  Amount and timing are public on Zcash.
- The **connected Base wallet is public on Base**, including the cashout itself.
- **NEAR 1Click and other route services still receive** destination, amount,
  timing and connection metadata — including on the shielded Buy path.
- **Payment apps receive what they need** for the final payment.

The site's own line is the correct summary:

> zcashto.cash must not be described as fully private or anonymous.

So the privacy is **partial and directional**: shielded in, transparent out, and
public on Base throughout. Zcash's strongest property does not survive the hop
into an EVM chain.

## It is not merchant checkout

This is stated three times, in the docs, the FAQ, and the schema:

> Can I pay someone with ZEC through zcashto.cash? **Not as a direct ZEC
> transfer.** The product converts ZEC value and creates a fiat cashout to the
> payment account selected in the flow.

> Does the recipient need a Zcash wallet? **No.**

This is the crucial limitation, and it is the same one that applies to Peer
generally: **there is no crypto-native payment.** A merchant receives **fiat**
from their payment app. ZEC — or BCH, or anything — never reaches the payee.

So "spend Zcash at a store" means "cash out to your own payment app, then spend
that balance." Two steps, and a KYC'd payment app in the middle.

## Liquidity, limits, and what is currently paused

Peer takers fill **from 50 USDC up to the available amount.** That is a real floor
and it matters for sizing.

**New Venmo Buy orders are paused.** Cash Out capabilities are listed as Revolut,
Mercado Pago, Zelle, Monzo, Alipay, and Chime — **Venmo is not currently among
them**, though Venmo Buy was supported before the pause. Availability, currencies
and liquidity change, and the site says the same thing on every page: **the live
form is authoritative**, not the documentation.

Fees are deliberately not published as fixed numbers. They are quoted per
request from the live form, and the site repeats this often enough that it reads
as a caution rather than an oversight.

## Referral and wallet sponsorship

Minor, but it shows the distribution model. Joining Peer with referral code
**`SHIELD`** supports the project. Wallet creation runs through **Privy** (passkey
or email, gas sponsored by zcashto.cash) or a browser wallet, where the user pays
Base gas.

The gas sponsorship is the tell that this is a consumer product: **someone is
absorbing a few cents per user to remove a step.** That is the cost of
distribution, not a technical detail.

## Recovery behaviour worth copying

One detail is better than most of our own wallet code:

> An unresolved route **resumes instead of being silently replaced.** The app
> blocks duplicate or uncertain submissions and keeps the payment-account
> identifier out of its persisted recovery record. Users should not resend,
> requote, or reuse an expired deposit address when a transaction is uncertain.

"Uncertain" is treated as a distinct state from "failed", and duplicate
submission is blocked rather than retried. That is precisely the class of bug our
own wallet has repeatedly hit — a failed run that leaves state half-advanced.

## What this means for BCH

Everything technical in this flow is **already verified working for BCH**:

| Step | zcashto.cash | BCH equivalent |
|---|---|---|
| Non-EVM source | ZEC → Base USDC via NEAR 1Click | **BCH → Base USDC, quoted and signed — verified** |
| One-time deposit address | per request | we already derive fresh per receive |
| Inventory model | pre-positioned, replenished in background | same |
| Fiat proof | Peer attestation | same |
| Payout | Zelle, Revolut, Monzo, Alipay, Chime | same |

**The BCH-specific gaps are narrow and known:**

1. **Privacy.** BCH has no shielded pool. Everything would be public on both
   chains, permanently. There is no version of the Zcash story to tell.
2. **One-time transparent address and UTXO selection.** Deriving a fresh address
   per request and spending it requires real UTXO management — which we have, and
   which is where our existing dust and change bugs live.
3. **The 50 USDC taker floor.** Real, and it sets a minimum practical size.

Everything else is assembly. **A `bchto.cash` is a front end, not a protocol** —
which is a much smaller project than the [x402 assessment](x402-on-bch) implied,
where BCH support would have meant writing a new chain integration into a
standard.

**Before building it:** the honest blockers are unchanged. Whoever runs the fiat
leg carries the money-transmitter question, and the chargeback exposure on
Zelle/Venmo is documented by Peer and unsolved. zcashto.cash's own privacy
disclosure is also the right template for what we would have to tell users.

## See also

- [[references/intents/index]] — the full path this product implements
- [[references/intents/near-intents-bch]] — the BCH route, quoted live
- [[references/intents/zkp2p-peer]] — the attestation and its trust assumptions
- [[references/receiving-an-address]] — our existing fresh-address-per-request
  behaviour, which this product depends on
- [[references/bch-zero-conf-security]] — what BCH's transparency costs, by contrast