---
pageType: reference
id: reference.wizardconnect.what-it-is
description: What WizardConnect is, who runs it, and why its transport is Nostr over NIP-59 rather than HTTPS — the piece that surprises everyone reading the spec for the first time.
sourceUrl: https://gitlab.com/riftenlabs/lib/wizardconnect
---

# What WizardConnect is

A bridge for **wallets that are not browser extensions**. It lets a dapp in a
browser ask a wallet on another machine — a phone, a desktop, a server — to sign
a transaction.

That sounds like WalletConnect. It is not, and the difference matters:

| | WalletConnect | WizardConnect |
|---|---|---|
| Designed for | EVM chains, one address | HD wallets, many addresses |
| Exposure | an xpub can be handed over | xpub **per chain**, dapp-derived pubkeys only |
| Sessions | long-lived, reusable | ephemeral, per pairing URI |
| Signing | arbitrary `eth_sendTransaction` | fixed sighash, structured request |

The reference implementation notes the risk it is designed around:

> connecting the entire HDWallet to a dApp exposes user to increased risk of
> compromising their seed. This risk is akin to giving the XPubKey to a third
> party.

## Who runs it

**Riften Labs** — the same people who run the Cauldron DEX. That is not a
coincidence: the first dapp to need it was a CashTokens swap page.

Published as three first-party npm packages:

```json
"@wizardconnect/wallet":     "0.2.3"   // WalletConnectionManager, WalletAdapter
"@wizardconnect/core":       "0.2.4"   // transport and protocol primitives
"@bch-wc2/interfaces":       "0.0.16"  // message shapes
```

`app.cauldron.quest` ships the dapp half in its own bundle — `signer.riften.net`,
`wizardconnect-session`, `sendSignRequest`, key exchange with a 30-second
timeout. It is a documented protocol, not an integration someone reverse-
engineered.

Reference wallets already working: Cashonize, Paytaca, ZapIt. Reference dapps:
Tapswap, Cauldron.

## The transport is Nostr over NIP-59

This is the part that catches people, and it is worth stating plainly because
the name suggests an HTTP API:

```js
// @wizardconnect/core dependencies
"nostr-tools": "^2.23.0"
"isomorphic-ws", "ws", "lossless-json"
import ... from "nostr-tools/nip59"     // NIP-59: a Nostr pubkey IS an http auth token
import ... from "nostr-tools/pool"
```

NIP-59 is the standard for **using a Nostr public key as a bearer token for an
HTTP endpoint**. The dapp hosts a relay; the wallet connects to it. So the
"connection" between a browser page and your wallet is a Nostr relay, with
sequence numbers, a message queue, ping/pong and reconnection handling.

Practical consequences:

- There is no server holding a key. Both sides are ephemeral identities.
- A dapp sees one Nostr pubkey per session, which is why its *unlinkability*
  matters — see [derivations](derivations-and-the-relay-key.md).
- The dapp can derive the addresses it needs from the xpubs in the handshake, so
  most requests do not even need a round trip.

## The interface a wallet implements

Small, and this is the good part:

```ts
interface WalletAdapter {
  walletName: string;
  walletIcon: string;
  getRelayPrivateKey(uri: string): Uint8Array;
  getPublicKey(path: DerivationPath, index: bigint): Uint8Array;
  getXpub(path: DerivationPath): string;
  signTransaction(request: SignTransactionRequest): Promise<{ signedTransaction: string }>;
}
```

Five methods. The library provides `WalletConnectionManager`, which handles
relay connection, key exchange, request sequencing and queuing. A wallet writes
the adapter and a UI.

Note the shape of the signing hook: `signTransaction` on the adapter is
**never called by the library**. Signing runs through the
`pendingSignRequest` event so the manager can queue it and a UI can approve it.
The reference implementation rejects the adapter method outright:

```ts
signTransaction: () => Promise.reject(
  new Error("signTransaction is handled via the pendingSignRequest flow")),
```

That is a small thing that is easy to get wrong and expensive to get wrong: an
adapter that signs directly would mean a dapp could trigger a signature without
any human seeing it.

## What the wallet receives

A `WcSignTransactionRequest` carrying:

- `transaction` — a raw unsigned hex, or a libauth transaction object
- `sourceOutputs` — the parallel array of UTXOs being spent, each with its
  locking bytecode, value, and for a contract input a `redeemScript` and
  `artifact`

`sourceOutputs` is transmitted with libauth's `stringify`, which serialises
`Uint8Array` and `BigInt` as `<Uint8Array: 0x…>` and `<bigint: …n>`. Parsing
that back is a ten-line reviver:

```ts
JSON.parse(json, (_k, v) => {
  if (typeof v === "string") {
    const b = v.match(/^<bigint: (?<b>\d*)n>$/u);
    if (b) return BigInt(b.groups.b);
    const u = v.match(/^<Uint8Array: 0x(?<h>[0-9a-f]*)>$/u);
    if (u) return hexToBin(u.groups.h);
  }
  return v;
});
```

`sourceOutputs` is currently required, and it should be: without it a wallet
cannot compute the sighash, and a wallet that signs without being able to
verify what it is signing is not a wallet.

## Related

- [Derivations and the relay key](derivations-and-the-relay-key.md)
- [The signing rules](signing-rules.md)
