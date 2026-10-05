---
pageType: reference
id: intents.near-intents-bch
description: NEAR Intents 1Click as the route for non-EVM assets — why a libauth/BCH wallet can be a payment source without a viem signer, the live 204-token asset list including BCH, and two signed quotes in both directions proving BCH -> Base USDC and back. Written 2026-10-05 from live API calls.
sourceUrl: internal/reference
---

# NEAR Intents: the bridge BCH actually rides

Peer settles in Base USDC. That is not where BCH enters. **NEAR Intents** is the
layer underneath that accepts non-EVM origins, and it carries BCH today.

This page records what was verified by calling the live API on 2026-10-05,
including two signed quotes. Everything here is reproducible.

## The problem it solves, precisely

x402-era infrastructure assumes an EVM chain with an account, a balance and a
signer that can move it. A BCH wallet has **no account, no balance, no nonce** —
it has UTXOs, and moving them means building and signing a transaction.

Peer Cash's own SDK documents this directly:

> NEAR Intents 1Click supports non-EVM origins such as Zcash, so the SDK does not
> pretend a viem wallet can execute the source transfer. It returns a signed quote
> with an origin-chain `depositAddress`; your wallet sends exactly once, then the
> SDK tracks the provider route into canonical Base USDC.

**That admission is the whole integration story.** The SDK stops pretending a
viem signer can do the work, and hands us a plain `depositAddress` on the origin
chain — which a libauth wallet can pay as an ordinary transaction, using code we
already have.

## The flow

```
1. quoteNearIntentsSource({ sourceAsset, amount, recipient, refundTo })
     → quote: depositAddress, depositMemo?, amountIn, deadline, ed25519 signature

2. wallet.send(depositAddress, inputAmount)      ← ours: one ordinary BCH tx

3. cash.submitNearIntentsDeposit({ depositAddress, depositMemo?, txHash })

4. cash.nearIntentsStatus({ depositAddress, ... })
     → track to SUCCESS, then reconcile the Base receipt

5. cash.cashout({ ... })                          ← Base-only from here
```

`tradeType: 'EXACT_OUTPUT'` matters: it lets the BCH amount be known **before**
the origin send, which is what makes the BCH-side leg deterministic instead of a
guess with a slippage band.

## Live verification

The token list is public and unauthenticated:

```bash
curl https://1click.chaindefuser.com/v0/tokens
```

**204 tokens across 35 chains** on 2026-10-05. The Bitcoin-family UTXO chains
are all there:

| chain | symbol | decimals | assetId |
|---|---|---|---|
| **bch** | **BCH** | **8** | **`nep141:bch.omft.near`** |
| btc | BTC | 8 | `nep141:btc.omft.near` |
| zec | ZEC | 8 | `nep141:zec.omft.near` |
| ltc | LTC | 8 | `nep141:ltc.omft.near` |
| doge | DOGE | 8 | `nep141:doge.omft.near` |
| dash | DASH | 8 | `nep141:dash.omft.near` |

**BCH is a first-class source asset.** Not bridged-to, not wrapped — a native
origin chain with a deposit address.

### Two live quotes

Both `POST /v0/quote`, both **HTTP 201**, both solver-signed `ed25519`.

**BCH → Base USDC** (exact output 1 USDC):

| field | value |
|---|---|
| `amountIn` | 328,628 sat (0.00328628 BCH) |
| `amountOut` | 0.999956 USDC |
| implied rate | ~$304/BCH |
| `timeEstimate` | **109 s** |
| `minAmountIn` | 318,769 sat (3% band) |
| `refundFee` | 500 sat |

**Base USDC → BCH** (exact output 1 BCH):

| field | value |
|---|---|
| `amountIn` | 326.56 USDC |
| `amountOut` | **1.0 BCH** |
| `timeEstimate` | **152 s** |
| `withdrawFee` | 500 (output units) |

**The second quote is Luke's flow, priced.** USDC arrives from the fiat leg, and
a signed quote converts it to BCH at a stated rate with a ~2.5 minute ETA.

## Getting the request right

Four things cost round-trips and are worth recording:

1. **`assetId` is lowercase.** A mixed-case Base USDC id returns
   `tokenOut is not valid`. The canonical form is
   `nep141:base-0x833589fcd6edb6e08f4c7c32d4f71b54bda02913.omft.near`.
2. **Required fields** the API will name for you on a 400: `recipient`,
   `recipientType` (`DESTINATION_CHAIN` | `INTENTS` | `CONFIDENTIAL_INTENTS`),
   `slippageTolerance`, `deadline`, `dry`.
3. **`refundTo` is validated as a real address per chain.** A placeholder CashAddr
   returns `refundTo is not valid`. Use a checksum-valid one.
4. **`refundType` is not the same enum as `recipientType`.** It takes
   `ORIGIN_CHAIN` | `INTENTS` | `CONFIDENTIAL_INTENTS` — passing
   `DESTINATION_CHAIN` fails.

The errors are descriptive, which makes this cheap to work with. Query
`/v0/tokens` for real asset ids rather than guessing them.

## What this does and does not solve

**Solved:** the BCH leg. A BCH wallet can be a source asset in a
fiat→crypto pipeline, receive a signed quote, send one ordinary transaction, and
track settlement — with no new cryptographic machinery and no bridging code we
write.

**Not solved:** everything upstream of USDC. Converting fiat to USDC requires a
regulated party, and that is where the KYC and money-transmitter problems live
(see [zkp2p-peer.md](zkp2p-peer.md)).

**The trust cost is real.** Between our BCH and the payer's fiat sit: Peer's
attestation service, the ZKP2P escrow, and a 1Click solver. Each is a party we
do not control. The BCH→USDC leg itself is signed and auditable, which is the
part we would verify.

## The right first step

**Do not build an onramp.** Build the BCH-side verifier and quote handler:

1. Accept an EIP-712 payment attestation, check it, release BCH. No licensing.
2. Wrap `quoteNearIntentsSource` → `submitNearIntentsDeposit` →
   `nearIntentsStatus` into the existing `run()` process model, so a quote
   failure and a wallet failure are distinguishable.
3. Test on **chipnet**, where a wrong route costs nothing.

Both quotes above were reproducible in four curl calls with no credentials. That
is the size of the integration — the hard part is deciding whether to be the party
that touches the fiat, and that is a business decision rather than a technical
one.

## See also

- [[references/intents/index]] — the full path
- [[references/intents/zkp2p-peer]] — the fiat leg and its three obstacles
- [[references/receiving-an-address]] — where the BCH lands
- [[references/in-wallet-swaps]] — an existing third-party router dependency, for
  comparison on what we already trust