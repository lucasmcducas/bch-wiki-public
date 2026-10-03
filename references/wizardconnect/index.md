# WizardConnect

How a wallet that is not a browser extension can still sign inside a BCH dapp:
the protocol, its derivations, the signing rules, and the specific things that
make it safe or unsafe to get wrong.

Written after building the wallet side of a WizardConnect connector, which is
the path that lets a Node wallet on one machine sign a swap built by the Cauldron
front end on another.

## Pages

1. **[The protocol](what-it-is.md)** — what WizardConnect is, who runs it, and
   why it exists. The transport is Nostr over NIP-59, which is the part that
   surprises people.
2. **[Derivations and the relay key](derivations-and-the-relay-key.md)** — the
   four chains, why the relay secret gets its own, and the assertions that keep
   it from leaking into a dapp's hands.
3. **[The signing rules](signing-rules.md)** — the fixed sighash, the two
   placeholder conventions, and the encoding bug that passes a superficial test.

4. **[Upgrading a crypto dependency](upgrading-a-crypto-dependency.md)** — capture golden signing vectors before the bump and diff the bytes after, because a green test suite cannot prove a signature is unchanged.

## Why this matters beyond swaps

The mechanism is general. Any BCH dapp that speaks WizardConnect can have a
transaction signed by this wallet, with the browser acting purely as a UI. The
swap is the first use, not the only one.

## The one-paragraph version

A dapp builds a transaction, leaves the wallet's inputs with an empty
`unlockingBytecode`, and relays the request over Nostr. The wallet verifies it,
asks the user, signs with `SIGHASH_ALL | SIGHASH_UTXOS | SIGHASH_FORKID` — so
the signature commits to the entire transaction and cannot be lifted onto a
different one — and relays the signed hex back. The dapp broadcasts. The wallet
never exposes a seed or a private key, only a per-session Nostr identity
derived on purpose from a chain the dapp never sees.

## Related

- [`../swaps/index.md`](../swaps/index.md) — the swaps section this came out of
- [`../entities/cashscript.md`](../../entities/cashscript.md) — why contracts need
  a pubkey and a signature as arguments
- [`../security/index.md`](../../security/index.md) — the swap investigation these
  pages sit inside

<!-- openclaw:wiki:wizardconnect:index:end -->
