---
pageType: reference
id: intents.zkp2p-peer
description: Peer/ZKP2P — how a Venmo or Zelle payment becomes a signed onchain attestation, via an AWS Nitro Enclave holding the authenticated session. Covers the custody split, the no-KYC claim, the three unsolved problems, and the verified absence of BCH. Written 2026-10-05.
sourceUrl: internal/reference
---

# ZKP2P: fiat payments as proofs

**Peer** (protocol name **ZKP2P**, `zkp2p.xyz` → `peer.xyz`) is a peer-to-peer
marketplace that lets a buyer pay **fiat** on a consumer app and receive
**USDC on Base** from an escrow contract. The interesting part is not the
marketplace; it is how it proves a payment happened on an app with no API.

Built by **P2P Labs Inc.**, from an Ethereum Foundation Privacy & Scalability
Exploratory grant project. Code is MIT and actively developed — `@zkp2p/cash`
0.7.1 and `@zkp2p/sdk` 0.14.6 were both published within days of this writing.

## The problem it actually solves

Buyer and seller are strangers. The buyer wants to pay with an app they already
have; the seller wants crypto. Neither wants to trust the other, and neither wants
a custodian.

> Buyer sends fiat **directly to the seller** on the seller's own payment app.
> Peer never touches it. The escrow releases crypto once payment is *proved*.

That is genuinely novel, and it is why the protocol is worth understanding even
if you never use it.

## How the proof works

Three generations, and the current one is not what the name suggests.

**V1 — zkEmail.** The buyer's payment app emails a receipt; DKIM signature
proves the email is authentic. Genuine zero-knowledge, and it only worked for
apps that email receipts.

**V2 — zkTLS.** Generic, composable, deployed and now legacy.

**V3 — TEE attestation. This is what ships.**

The buyer completes the payment in the real Venmo or banking app. Then an
**AWS Nitro Enclave** holds the authenticated session, looks up the payment, and
signs an **EIP-712 `PaymentAttestation`** offline. Onchain,
`UnifiedPaymentVerifierV3` checks method, currency, **payee, amount, timestamp,
intent hash**, and nullifiers against the intent, then escrow releases.

Two consequences worth being precise about:

- **The enclave is not trusted to be honest about what it saw.** It is trusted to
  run the code it claims. The key is KMS-wrapped and **PCR8-gated**, and clients
  can verify the running enclave via `GET /attestation?nonce=...`.
- **This is a signed statement, not a zero-knowledge proof.** V3 abandoned zkTLS
  for TEEs. Peer claims it made verification ~100× faster (~30s → sub-second).
  The privacy claim is that the *input* is minimised (amount, recipient,
  timestamp), not that the verifier is untrusted.

### The Venmo trick

Venmo has no usable third-party payment API. Peer gets around it with
**optional receipt linking**: the user authorises a **Gmail or personal Outlook
inbox** that receives their Venmo receipts, a hosted Google flow reads it, and
that produces the attestation. Linking requires no wallet and no deposit, and
`cashout()` never requires it.

So "Venmo has no API" is true and still not the end of it — but the cost is
granting inbox access, which is a large privacy surface that the user is asked
to accept in exchange for convenience.

## Custody: the split that makes it work

| Layer | Trust |
|---|---|
| Fiat | **Nobody.** Moves directly between the two humans on the payment app. |
| Escrow | **Non-custodial.** Smart contracts on Base; Peer holds no keys. |
| Attestation | **A single trusted operator** — Peer's own enclave service. |

Peer is explicit that it is "not an exchange, broker, or custodian." That is
true of the money. It is *not* true of the verification, and Peer's own risk
page says so:

> Payment verification currently runs through attestation infrastructure operated
> by the Peer team (**comparable to single-sequencer L2s today**). A compromised
> or colluding verifier could in theory attest to payments that didn't happen.

**That sentence is the honest summary of the whole design.** On-chain custody is
trustless; the off-chain oracle of "did this person pay" is one company. Anyone
describing this as "trustless fiat payments" is describing half of it.

## What is verifiable, and what is not

A recipient can prove: a signed EIP-712 attestation checked onchain, containing
payment ID, amount, currency and timestamp — plus the escrow release txid.

A recipient **cannot** prove: **who paid.** On-chain observers see amounts and a
hashed payee; the off-chain identity stays with the payment app. "Verify payment,
not identity" is accurate as a privacy property and should not be read as
"cryptographically proven."

## KYC

**Explicitly none.** "There is no KYC on the protocol — your identity lives with
your payment platform, not on-chain."

The ToS does add sanctions screening — users represent they are not
OFAC-sanctioned. That is a contractual representation, not identity verification.

This is the interesting part for us: *no KYC* means no custodial account, but it
also means **Peer cannot be the regulated party.** Which leads directly to the
obstacle below.

## BCH: verified absent

Not an inference — checked across the `@zkp2p/contracts-v2` tarball, the full doc
corpus (`llms.txt` plus all 74 sitemap URLs), and the ecosystem page:

- `bitcoin cash`, `BCH`, `cashaddr`, `bitcoincash`: **0 hits**
- Peer's escrow asset: **USDC on Base** (`0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`)
- Destination assets are **bridged** — Ethereum, Solana, Hyperliquid, and Zcash
  via NEAR Intents

**BTC is not BCH.** "Bitcoin" in Peer's marketing means a bridged destination
asset. Do not let a `grep bch` collide with **Zcash** either — both were checked
separately, and Zcash is genuinely supported while Bitcoin Cash is not.

The BCH route exists one layer down, in the SDK's NEAR Intents source list. See
[near-intents-bch.md](near-intents-bch.md).

## Three obstacles, none of them engineering

**1. Zelle is contractually P2P-only.** Discover's and Wells Fargo's Zelle
addenda: *"The Service is intended for personal, not for business or commercial
use… we reserve the right to suspend or terminate your use."* No public API
exists. Jack Henry and US Bancorp offer Zelle APIs, but they are contract-gated to
financial institutions. Zelle shut its standalone app in 2025 — it now lives
inside bank apps. Any Zelle-based product is scraping an inbox and hoping.

**2. Reverse-payment risk is unsolved, and Peer admits it.** Venmo and PayPal are
precisely the two methods that carry non-zero onchain risk windows, so they get
stake-backed dispute protection; Cash App is non-chargebackable and stays public.
But:

> Staking cannot prevent a payment provider from reversing fiat.

A buyer can take the crypto and reverse the payment. Staking deters; it does not
prevent. For a **$20** payment, staking worth more than $20 is not a real control.

**3. FinCEN money-transmitter exposure.** Accept a payer's fiat and hand them
crypto from a pool, that is money transmission under FinCEN's CVC guidance
(FIN-2019-G001). The "a user buying crypto with their own money is not an MSB"
carve-out does not cover taking someone else's fiat and delivering crypto.
Peer is not exposed because it holds no funds — **a product that does the
merchant role is.**

## Maturity

Actively built, genuinely small. Contracts MIT, 26★, 344 commits, pushed two days
before this writing. The v1 monorepo is 340★ and archived. Six contributors
total on the contracts repo, one with 243 commits. Sherlock audits are claimed
for V2 and V3; **the reports were not linked on the risks page and I did not
verify them.** A 2024 disclosed bug (underconstrained ZK circuits, verifiers
disconnected) was found via search snippet only.

## Verdict for a BCH wallet

**Do not integrate with Peer for the fiat leg.** It is a real protocol with real
contracts, but it settles in Base USDC, is small, centralises exactly the step
that matters (payment verification), and carries a chargeback exposure it
documents rather than solves.

**Do use the pattern.** The design worth copying is the *receiver-side verifier*:
a component that accepts a signed attestation, checks it against a locally held
intent, and releases funds. It is small, has no licensing burden, and is the only
piece we would have to own.

If the fiat leg is ever built, the honest options are PayPal's "Pay with Venmo"
(webhook-confirmed, merchant KYC, terminates in a PayPal account — so *we* are
the regulated party), or Peer (no KYC, but we take the chargeback risk and depend
on a 6-person company for proof).

- [[references/intents/zcashto-cash]] — a consumer product built on this
  protocol end to end, and the clearest statement of what it does not do

## See also

- [[references/intents/index]] — the full path this page sits in
- [[references/intents/near-intents-bch]] — where BCH actually enters
- [[references/wizardconnect/what-it-is]] — the signing side we already own
- [[security/wallet-threat-model]] — custody questions raised by adding an escrow