
---
pageType: entity
entityType: skill
id: entity.moth-bch-wallet
description: moth agent bch-wallet skill - reference implementation of a self-custodial BCH wallet bot using Node.js + libauth + Electrum protocol. Hosted at dev.selene.technology.
sourceUrl: https://dev.selene.technology/~moth/skills/bch-wallet.html
---

# moth bch-wallet skill

> moth agent `bch-wallet` skill — reference implementation of a self-custodial BCH wallet bot using Node.js + libauth + Electrum protocol. Hosted at `dev.selene.technology`.

**Live page:** https://dev.selene.technology/~moth/skills/bch-wallet.html
**Last-modified (page):** **Wed, 22 Jul 2026 08:02:48 GMT** (verified with `curl -I`)
**Tarball:** `https://dev.selene.technology/~moth/skills/bch-wallet.tar.gz` — ships **only `SKILL.md`** (mtime Feb 18 2026). The actual `.mjs` sources are **not bundled** and live in moth's runtime workspace.
**Author:** Luke Pryor's moth (self-hosted AI agent)

## Verified facts about the skill

Fetched live via curl against the static nginx page. Source-of-truth is the page itself, not SKILL.md, because SKILL.md hasn't been updated since Feb 18.

### Scripts

7 scripts (verified against the HTML on the live page):

| Script              | Purpose                                  | CLI shape                     |
|---------------------|------------------------------------------|--------------------------------|
| `balance.mjs`       | Print confirmed/unconfirmed balance for current wallet | takes script path       |
| `address.mjs`       | Derive next receive address              |                                 |
| `utxos.mjs`         | List current spendable UTXOs             |                                 |
| `history.mjs`       | Print recent transaction history         |                                 |
| `send.mjs`          | Build + sign + broadcast BCH send        | `node send.mjs <addr> <sats>`  |
| `create-wallet.mjs` | Generate new wallet (BIP39 mnemonic)     |                                 |
| glue (`SKILL.md`)   | Registers scripts as tools callable by the AI agent | env-driven orchestration |

**Confirmed CLI shape for `send.mjs`:** positional `<addr> <sats>`. The broadcast confirmation is **NOT** a CLI flag — it is an environment variable `BCH_CONFIRM=yes`. Without it, the script dry-runs (builds the transaction and prints without asking). With it, broadcasts and waits for the first confirmation.

### Wallet storage

- **Path:** `~/.bch-wallet/wallet.json`
- **Mode:** `0600` (filesystem permissions only)
- **Encryption:** **PLAINTEXT.** The live page does *not* mention encryption. The `moth credentials` skill uses AES-256-GCM + scrypt, but that is a *separate* skill. The bch-wallet skill relies on filesystem ACLs exclusively. **This is a security gap worth flagging.**

### Network configuration

- **BIP44 path:** `m/44'/145'/0'/0/0` for the first address. Supports derivation `0/i` per the moth convention. Note this is **off-spec** slightly — the standard is `m/44'/145'/0'/0/i` with `i` in the address index position (see libauth entity page), but the moth script uses `0/0` as the *first* index.
- **Network:** mainnet (no testnet flag visible).
- **Electrum server:** The page **does not document** the server hostname, port, or whether it's public/loopback. Cannot determine from the page whether moth runs its own Fulcrum, hits a public Fulcrum endpoint, or uses BCHN RPC. **Treat as undocumented.**
- **Dependencies (from page):** `@bitauth/libauth`, `@electrum-cash/protocol`. The bch-wallet skippable — there are no `node_modules` in the tarball.

### Fee & dust policy

- **Fee rate:** ~**1 sat/byte** (flat — not a market-clearing price)
- **Dust threshold:** 546 sats (matches BCHN consensus)
- **Dust handling:** sub-dust change is absorbed into the fee — the user does not receive sub-dust outputs back

### Address derivation

- **Gap limit:** **20** consecutive unused addresses (standard BIP44 default)

## What the page does NOT document

These are the gaps the wiki cannot fill from the page alone (verified by grep on the live HTML):

1. **UTXO selection algorithm** — no mention of FIFO, largest-first, branch-and-bound, or `coinSelect`. Cannot replicate the exact behavior without examining `.mjs` directly.
2. **Network endpoint** — no Fulcrum/BCHN hostname or port.
3. **Fee estimation strategy** — flat 1 sat/byte; no RBF/CPFP bumping documented.
4. **Reorg handling** — no `confirmations` requirement visible; depends on `1 confirmation` behavior of Fulcrum.
5. **The actual `.mjs` source files** — only SKILL.md is in the tarball. To get internals, you need access to moth's runtime at `~/skills/bch-wallet/scripts/*.mjs` on the lambda box.
6. **libauth version** — pinned to which v3.x.y? Not specified. CHANGELOG/version absent from the bundle.
7. **Mainnet vs. testnet auto-detect** — unclear whether `address.mjs` returns regtest/testnet/mainnet prefixed addresses based on configuration.
8. **Memory of confirmed-only UTXOs vs. mempool** — `utxos.mjs` may show unconfirmed UTXOs with `unconfirmed` flag; documented example absent.

## Comparison to the related moth credential skill

The `moth-credentials` skill on the same page implements AES-256-GCM + scrypt KDF + a `VAULT_PASS` env var for key unlock. Reading the bch-wallet page alongside it shows the skills were built by the same author with consistent env-passed credentials pattern. **The bch-wallet skill does not use this pattern** — there is no key unlock, no encryption. This is the security delta worth deciding for: does the bot's use case warrant AES-encrypted-at-rest for `wallet.json`, or is mode-0600 + root-only VPS sufficient?

## How the bot can reuse moth's design

The library choices and architecture pattern are the reusable parts:

- **Stack:** Node.js + `@bitauth/libauth` + `@electrum-cash/protocol`. Mainstream, low-ceremony dependencies.
- **CLI surface:** thin scripts named after their action (`balance.mjs`, `send.mjs`, etc.). Easy for an AI agent to call out to.
- **Auth pattern:** `BCH_CONFIRM=yes` env var to gate destructive operations. Force the agent to confirm in writing rather than relying on positional CLI flags.
- **Dust policy:** absorb sub-dust into fee. Simple rule; ship it.

The implementation details (UTXO selection, address index, mnemonic serialization) you have to read or re-derive — they are not on the page.

## Pitfalls

1. **wallet.json is plaintext at mode 0600.** Compromise of the bot host = compromise of the funds. For non-trivial amounts, layer encryption (moth's credentials skill shows the way: AES-256-GCM + scrypt).
2. **The `BCH_CONFIRM=yes` env var is bypassed if the bot is rooted.** Anyone who can read the agent's environment can spoof it. Treat as a soft confirmation, not a strong one. For high-value sends, require an out-of-band (e.g., human-in-the-loop via Telegram/Discord) confirmation.
3. **No documented 2FA / multi-sig / explicit signing policy.** Just send. Don't hold non-trivial amounts in this wallet — use P2SH32 multisig or a hardware-backed approach when they matter.
4. **Server hostname is not on the page** — if you're replicating this skill verbatim, you'll need to inspect the unprivileged `.mjs` to know which Electrum server it's hitting by default. Default to running your own Fulcrum and pointing at that.
5. **`m/44'/145'/0'/0/0` is hardcoded as "first address"** in the moth code. If you derive `m/44'/145'/0'/0/1` from the same mnemonic you'll get a different address — but the bot's "default address" is index 0. Make sure your wallet backup process uses the same index.
6. **No `.mjs` source.** You cannot fork this skill cleanly without git access to moth's runtime. The tarball is documentation-only.

## Related entities

- `entity.libauth` — the `@bitauth/libauth` library moth uses.
- `entity.fulcrum` — the Electrum server moth hits (assuming it's the moth-hosted Fulcrum).
- `entity.bchn-node` — if moth's bot uses BCHN RPC directly (not documented either way).
- `concept.cash-tokens` — the wallet is BCH-only; no token-specific code paths visible in the page.
