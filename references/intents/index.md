---
pageType: section
id: intents.index
description: Fiats, proofs, and intent-based routing — the ZKP2P/Peer protocol that turns a Venmo or Zelle payment into a signed onchain attestation, and NEAR Intents, which routes Bitcoin Cash across the gap. Written 2026-10-05 from primary sources plus live quotes against both APIs.
sourceUrl: internal/synthesis
---

# Intents

Two different things get called an "intent", and this section covers both because
the useful flow runs through them in series.

The question that starts here: **can someone pay us with Zelle or Venmo and have
BCH arrive in our wallet?** Not "can they buy BCH" — the rails for that exist.
Can an *invoice* be created, paid in a consumer app, and settled in BCH with
proof the payment happened.

The short answer, verified live on 2026-10-05: **the transport works and is
priced. The obstacles are licensing and chargeback risk, not engineering.** A
full path was quoted end to end in both directions.

## Pages

1. **[ZKP2P: fiat payments as proofs](zkp2p-peer.md)** — what Peer/ZKP2P is, how
   a Venmo payment becomes an EIP-712 attestation signed inside an AWS Nitro
   Enclave, and the custody split that makes it work. Includes the honest part:
   verification is a single trusted operator, and Peer says so.

2. **[NEAR Intents: the bridge BCH actually rides](near-intents-bch.md)** — why a
   non-EVM, non-viem asset can be a payment source at all, the live asset list,
   and two real signed quotes in both directions.

3. **[zcashto.cash: the precedent, working](zcashto-cash.md)** — the consumer
   product that completes this flow for Zcash, and the closest working template
   for a BCH version. Read this one if you are deciding whether to build.

## The whole path, in one block

```
person pays Venmo / Zelle on their own app
        │
        ▼
ZKP2P attestation service  ── EIP-712 PaymentAttestation, signed in an
        │                     AWS Nitro Enclave holding the session
        ▼
UnifiedPaymentVerifierV3 ── checks payee, amount, timestamp, intent hash,
        │                     nullifiers. USDC released from Base escrow.
        ▼
NEAR Intents 1Click      ── quote + deposit address, non-EVM source supported
        │
        ▼
our BCH wallet
```

Every hop below the fiat app is a signed or onchain artefact. The fiat hop
itself is not, and cannot be — which is where the trust problem lives. That
distinction is the subject of [the attestation page](zkp2p-peer.md).

## What is verified, and what is not

Verified by direct query on 2026-10-05:

- **A live BCH → Base USDC quote**: 328,628 sat in, 0.999956 USDC out, ETA 109s,
  solver-signed `ed25519`. HTTP 201.
- **A live Base USDC → BCH quote**: 326.56 USDC in, 1.0 BCH out, ETA 152s,
  signed. HTTP 201.
- **1Click carries BCH**: `nep141:bch.omft.near`, 8 decimals, among 204 tokens
  across 35 chains.
- **Peer supports Venmo, Zelle, Cash App and PayPal** in its production method
  registry.

Could **not** verify, and stated as such throughout: any BCH figure in Peer's own
token list (there is none — see below), Peer's audit reports, funding amount, and
production volume. The Venmo/Zelle receipt-reading flow was **read from
documentation, not executed** — it requires a real payment and a real inbox.

## The finding that matters most, and it is a negative

**Peer itself does not support BCH.** This is a thorough verified negative, not
an inference: zero hits for `bitcoin cash`, `BCH`, `cashaddr` and `bitcoincash`
across the `@zkp2p/contracts-v2` tarball, the full documentation corpus
(`llms.txt`, all 74 URLs in the sitemap), and the ecosystem page. Peer's escrow
settles in **USDC on Base**; its destination assets are bridged EVM and Solana
tokens.

Do not read that as "the flow is impossible." The distinction is what this
section is about:

- **Peer's own asset list** → Base USDC. BCH absent.
- **The SDK's source-route list** → `NEAR Intents`, which supports non-EVM
  origins, and BCH is in it.

Both are true at once, and stopping at the first is how you conclude the answer is
"no" when it is actually "yes, one layer down." Zcash was the first non-EVM asset
wired this way, and Peer announced it explicitly.

## Where to start, if you are building this

Not the onramp. The onramp is a payments business — FinCEN money-transmitter
exposure, KYC, and a settlement window in which you hold customer fiat.

The cheapest defensible piece is the **recipient-side verifier**: something that
accepts a signed payment attestation, checks it, and releases BCH. It is the only
component with no licensing burden, it is testable on chipnet today, and it
composes with Peer and ZKP2P instead of competing with them. It also survives the
obvious failure modes — a Zelle policy change, or Venmo's BCH listing moving —
which anything built on scraping an app's inbox would not.

See [zkp2p-peer.md](zkp2p-peer.md) for the three obstacles that must be solved
regardless of who builds them, [near-intents-bch.md](near-intents-bch.md) for
the routing layer that already works, and
[zcashto-cash.md](zcashto-cash.md) for the whole thing assembled into a consumer
product for Zcash — which is the template, and the reason this reads as a front
end rather than a protocol.

## See also

- [[references/receiving-an-address]] — the BCH-side address transports, and why
  a QR beats a retyped string
- [[references/wizardconnect/what-it-is]] — the existing dapp-signing half, and
  the human-approval rule an automated flow would have to respect
- [[security/wallet-threat-model]] — the custody questions raised by introducing
  an escrow and an attestation service into the path
- [[references/in-wallet-swaps]] — where our own swap path already relies on a
  third-party router, for comparison on trust assumptions