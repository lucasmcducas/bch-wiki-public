---
pageType: reference
id: reference.security.key-custody-and-oracle
description: Custody and oracle-layer findings from a 2026-10-02 audit of bch-bot — no key zeroisation, unbounded gap-limit scans, a fabricated token category constant, no amount bounds on CashTokens, no cross-node quorum, and two libauth versions in one tree. Separates what is exploitable today from what is theoretical.
sourceUrl: internal/audit
---

# Key custody and oracle trust in bch-bot

*Written 2026-10-02 from a read-only audit of the live code at `~/bch-src` on the
Omarchy box (git `d63bda9`), plus the file-permission state of the mainnet
wallet directory. Complements
[`wallet-key-storage.md`](wallet-key-storage.md) (what production wallets do at
rest) and [`wallet-threat-model.md`](wallet-threat-model.md) (threat classes).
**This page covers the bot's own code, not the ecosystem.***

Findings are ranked by realistic exploitability against a wallet holding real
BCH, and each is labelled **exploitable today** or **theoretical**.

---

## 1. No zeroisation of key material — theoretical, but free to improve

**Exploitable today: no.** libauth is pure JS; every `Uint8Array`/`Buffer` holding
a private key or the seed is ordinary heap memory with no protection, and JS gives
no allocator control. A grep for any wipe primitive across `lib/` and `scripts/`
returns nothing:

```
$ grep -rn "fill(0)\|zeroi\|wipe" --include=*.mjs lib/ scripts/ | grep -v test
(no results)
```

This is a **known and largely unfixable** limitation rather than an oversight, and
the honest framing is the one from the primary write-up: *"You can't wipe a secret
in JavaScript, but you should try."*[^nop33] The runtime may have copied the
buffer; GC timing is not under program control; a zeroed buffer is
best-effort only.

What *is* under program control, and is not done today:

- `lib/wallet.mjs:156` — `deriveSeedFromBip39Mnemonic(w.mnemonic, …)` creates the
  64-byte seed and **never zeroes it**.
- `lib/wallet.mjs:204` — `deriveAddress` returns `{ address, pubkeyHash,
  privateKey }`. The private key travels back to every caller in an object that
  then gets spread into the next structure (`send.mjs:137`,
  `send-token.mjs:249`).
- `lib/sign.mjs:74` — a fresh `privateKey` per input, held for the process
  lifetime of a command that may sit on a network round-trip.

**Judgement, not an established standard:** the real reduction available in JS is
**reducing lifetime and copies**, not wiping. Concretely: stop returning
`privateKey` from `deriveAddress` (it is never needed — every call site
re-derives via `deriveChildPrivKey`), and run the process in a short-lived
subprocess so the heap is destroyed on exit. Today `bch-bot` is a Node process
that loads the seed, holds it, and only exits when the command finishes — so the
window is already short. **I rate this low.** Say it is not a clean bill of
health: a swap connects to a third-party WebSocket while the seed is live in the
same heap, and that is the scenario worth thinking about.

---

## 2. The seed is loaded before the network is touched — a correct ordering worth keeping

`lib/wallet.mjs` has no network imports at all; `loadHdNode()` is pure
derivation. Every script calls `loadWallet()`/`loadHdNode()` and *then*
`connect()`. So a hostile Electrum server cannot observe a connection made before
the key is in memory.

Two scripts do invert this in spirit:

- `bin/bch-bot` does **not** set `BCH_WALLET_DIR` itself — it only
  `spawn(process.execPath, …, { env: process.env })` (`bin/bch-bot:105`). The
  *shim* at `~/.local/bin/bch-bot` exports it. This is documented in
  `failure-modes-we-hit.md` §22 ("running a script directly queries a different
  wallet"), and it is still an extra indirection: two files define where the
  wallet lives.
- `sweep.mjs:95` and `stake.mjs:98` gate on
  `process.env.BCH_WALLET_DIR === undefined` to decide whether the wallet is
  "mainnet by default". Setting the variable is what *disables* the mainnet check.
  That is backwards for a safety property: the variable should never be able to
  weaken a check.

---

## 3. A fabricated constant in the token registry — exploitable, low blast radius

**Exploitable today: yes, but only cosmetic.** `lib/token-registry.mjs:29`:

```js
'2b2c7c0b4bd3b0f1f3f7d3b1d2a5e6c7d8e9f0a1b2c3d4e5f60718293a4b5c6d':
  { symbol: 'BRAINROT', decimals: 0, name: 'Brainrot Battles' },
```

This is **not a real category id**. It is 64 hex characters of ascending
sequence — `c7d8e9f0a1b2c3d4e5f6`, `1b2c3d4e5f607` — with a repeating tail. It is
the same fabrication shape as `failure-modes-we-hit.md` §5 ("a real-looking
prefix followed by an invented tail"), and it has the same property: it is
indistinguishable from a real hash to a reader, so nobody challenges it.

Compare the two rows beside it, which are real: PUSD
`2469acc5…a02544` and ROACH `892cef80…0135ff53` — no structure, no runs.

Consequences, in order of severity:

1. **`decimals: 0` is a live mis-render.** `send-token.mjs:128` reads decimals
   from this registry to decide whether an amount is a display amount or base
   units. Any UTXO on the fabricated category — which cannot exist — would render
   wrong. The blast radius is genuinely small *because the category cannot exist*.
2. **It is not verified against anything.** No test asserts the row against an
   external source; `grep -rn BRAINROT scripts/` finds only the definition. It is
   exactly the "constant with no callers is either dead code or an unfinished
   check" case from §5.

**Rule (generalising §5):** read category ids from the published artifact or
omit them. A registry whose whole purpose is to be *right about other people's
tokens* must not contain a guessed entry. **Fix: delete the row.**

---

## 4. No CashTokens amount or category bounds — exploitable, but fails closed

The spec is explicit[^ct-spec]:

> "Amount — The number of fungible tokens held in this output (an integer between
> 1 and 9223372036854775807)."

and

> "`<category_id>` – After the PREFIX_TOKEN byte, a 32-byte Token Category ID is
> required."

Neither bound is enforced anywhere in this codebase:

```
$ grep -rn "9223372036854775807\|2\*\*63\|MAX_TOKEN\|length !== 64" --include=*.mjs lib/ scripts/
(no results)
```

`createTokenOutput` (`lib/tokens.mjs:58`) takes `category` and `amount` and hands
them to `hexToBin` and `BigInt` respectively. `hexToBin` (`lib/hex.mjs:8-17`)
validates only that the input is an even-length string:

```
$ hexToBin("abcd")          -> 2 bytes   (no throw)
$ hexToBin("ab".repeat(31)) -> 31 bytes  (no throw)
$ hexToBin("ab".repeat(33)) -> 33 bytes  (no throw)
```

`send-token.mjs` passes a user-typed category straight through
(`:113` `.toLowerCase()` → `:163` `category: targetCat`) with no length or hex
check, unlike `lib/router.mjs:27` which *does* validate
`/^[0-9a-f]{64}$/i` at the swap boundary.

**Honest severity: low, and it fails closed.** Both cases end inside libauth's
encoder, which throws on a malformed category or an out-of-range amount, and the
transaction is never signed. The real risk is a *confusing* failure far from the
cause — the `failure-modes-we-hit.md` §-pattern — plus the fact that a 33-byte
category would be a silent category change if any code path ever accepted one.
**Fix is cheap and belongs in `createTokenOutput`/`createNftOutput`:**

```js
if (!/^[0-9a-fA-F]{64}$/.test(category)) throw new Error(`category must be 64 hex chars, got ${category.length}`);
if (amount < 1n || amount > 9223372036854775807n) throw new Error('FT amount out of CashTokens range');
if (commitment && commitment.length > 80) throw new Error('NFT commitment exceeds the 40-byte maximum');
```

The commitment bound is also unenforced (`lib/tokens.mjs:93` pads an empty
commitment to 32 zero bytes, which is legal, but nothing stops a 100-byte one).

---

## 5. One untrusted node is the sole source of truth — exploitable, low probability

Every read goes through `connect()` (`lib/network.mjs:62-96`), which returns the
**first server that accepts a socket** and never cross-checks a second. For a
mainnet wallet the first entry is `rostrum.cauldron.quest:50004` — a server run
by the same operator as the DEX router.

What a malicious Electrum/Rostrum server can do is well characterised in the
literature, and the standard statement is that it **cannot forge a signature or
steal a key** — it can only lie and withhold:

> "There are no 'authorized servers'. By design, they cannot interfere with
> bitcoin transactions made by clients except: 1) lie about account balances and
> 2) not relay a valid transaction to the rest of the network."[^electrum-authority]

Concretely, against this codebase:

| What a node could lie about | Effect here | Bounded? |
|---|---|---|
| UTXO `value` | fee/change arithmetic uses a wrong input total (`send.mjs:40`, `sweep.mjs`) | **No** — `signP2pkhTransaction` commits to the *real* value in the preimage, so a wrong value produces a signature the network rejects, not a loss. The documented analogue is that a lying server can push the wallet toward overpaying fees[^beignet]. |
| Missing UTXOs | understates balance | Yes — cosmetic |
| `has_token`/`token_amount` | wrong token balance | Yes — `normaliseTokenData` (`lib/tokens.mjs:236`) under-reports rather than over-reports, by design |
| `blockchain.transaction.broadcast` returning `{}` | **reported as success** | **No** — see below |
| Refusing to relay | swap never confirms | Yes — funds stay in our inputs |

**The `{}` case is the live one.** `send.mjs:164-168`:

```js
const result = await client.request('blockchain.transaction.broadcast', tx_hex);
if (typeof result === 'string' && result.startsWith('Error')) { throw ... }
console.log(JSON.stringify({ ...outJson, broadcast: true, server_response: result }, null, 2));
```

A `{}` response passes the `startsWith('Error')` test and is printed as
`"broadcast": true`. `failure-modes-we-hit.md` §16 names exactly this — *"a
`broadcast` that returned `{}` was logged as `BROADCAST ACCEPTED`, when no
transaction had been broadcast at all"* — as a *past* finding. **It is still
present in `send.mjs`, `send-token.mjs:285`, `sweep.mjs:182`,
`add-liquidity.mjs:267`, and `round-trip.mjs:142`.** The fix already exists in
`lib/router.mjs:583-591` (`broadcastViaElectrum` validates the txid shape) and was
never applied to the four sibling scripts.

That is a **confirmed-not-yet-fixed instance of a documented bug class**, and it
is the single cheapest high-value fix in the repo:

```js
// replace the startsWith('Error') test in all five scripts with:
if (typeof result !== 'string' || !/^[0-9a-fA-F]{64}$/.test(result)) {
  throw new Error(`broadcast returned an unexpected response: ${String(result).slice(0,200)}`);
}
```

**Cross-checking strategy (cheap, partial).** For a wallet this size, a quorum of
two independent servers on the UTXO set is practical: query both, and refuse on
disagreement rather than picking one. The repo already has a failover *list*;
what it lacks is *comparison*. Note that most entries in `SERVERS.mainnet`
(`lib/network.mjs:14-31`) are Fulcrum nodes, and only the first is
CashTokens-aware — so a naive two-server quorum would **hide token balances**
(see `failure-modes-we-hit.md` §23). Any quorum must compare only plain BCH UTXOs,
or must be token-aware on both sides.

---

## 6. Gap-limit scans are sequential and unbounded — local DoS

Every read path derives 20 receiving + 20 change addresses and issues **one
`listunspent` round trip per address, sequentially** (`send.mjs:31-33`,
`sweep.mjs:35-48`, `swap.mjs:156-198`). 40 round trips at Electrum latency is
several seconds per command. It is correct, and it is the standard design — but
there is no cap on `deriveReceivingAddresses(n)` from the CLI (§5 of
[`pre-signing-invariant.md`](pre-signing-invariant.md)) and no concurrency limit.

**Not remotely exploitable.** Electrum servers are TLS-wrapped and the client
opens one connection per `connect()`. The realistic risk is a hostile server
answering slowly to stall a command while the seed is in memory — which connects
to §1: the exposure window is longer than the operation needs.

---

## 7. Supply chain: pinned, integrity-checked, with one wart

**This category is genuinely in good shape. Reporting it as such.**

`package.json` pins **exact** versions with no `^` or `~`:
`@bitauth/libauth` `3.0.0`, `@cashlab/cauldron` `1.0.3`, `@cashlab/common`
`1.0.5`, `@electrum-cash/network` `4.3.0`, `@electrum-cash/web-socket` `4.0.3`.

`package-lock.json` is `lockfileVersion 3` with **20 entries, every one carrying
both `resolved` and `integrity`** — verified by reading the file, not assumed.
That is the control that matters: `npm ci` enforces it, and a hijacked release
inside a semver range cannot enter the tree because the integrity hash would not
match. This is the exact scenario the lockfile exists to stop — the Sept 2025
`chalk`/`debug` hijacks and the Shai-Hulud worm both spread through ranges the
lockfile pins shut.[^lockfile]

**The wart: two versions of libauth in one tree.**

```
node_modules/@bitauth/libauth                                3.0.0
node_modules/@cashlab/common/node_modules/@bitauth/libauth    3.1.0-next.6
```

The signing code imports the top-level `3.0.0`. `@cashlab/common` carries its own
`3.1.0-next.6` — a **pre-release**, and the build-pattern skill explicitly warns
*"Avoid `v3.1.0-next.x` — pre-release, API drift."*[^build-pattern] It is only
reached through `@cashlab/cauldron`, so today it never signs anything. But it is
worth recording as a latent hazard: a future `npm update` that hoists
`3.1.0-next.6` to the top level would change **the signing library** without a
single code change. Pin it with an `overrides` block:

```json
"overrides": { "@bitauth/libauth": "3.0.0" }
```

**No upstream audit exists to lean on.** libauth markets itself as
zero-dependency and "better auditability"[^libauth-npm], but no third-party
security audit was found for it, and no public CVE either. That is the same
"documented, not verified" position
[`dex-swap-integration.md`](dex-swap-integration.md) records for the router.
Per the index's own executive-summary point 12: *"No public CVE exists for any
production BCH wallet — do not read that as a clean bill of health."*

---

## 8. File permissions and the accepted plaintext risk — verified still intact

The documented accepted risk is a plaintext seed at rest. **Confirmed, and the
permissions are correct.**

```
$ stat -c "%a %U:%G %n" <wallet dir>/*
600 luke:luke wallet.json
600 luke:luke state.json
$ stat -c "%a %n" <wallet dir>
700
```

Both files `0600`, directory `0700`. `lib/wallet.mjs:110,113,219,235` re-apply
`chmodSync(…, 0o600)` after every write, which defends against the process
umask — relevant because **this host's umask is `0022`**, so a
`writeFileSync` without the explicit chmod would land as `0644`.

**The `wallet.json.v1.bak` backup does not exist.** The task brief described it as
a known plaintext copy. It is not present in the mainnet directory, and
`encrypt-wallet.mjs:137-139` creates it only when encrypting:

```js
const backupPath = WALLET_FILE + '.v1.bak';
copyFileSync(WALLET_FILE, backupPath);
```

**So the residual risk is conditional:** the moment anyone runs
`bch-bot encrypt-wallet`, a full plaintext copy of the seed is written next to the
encrypted wallet, and the script **never `chmod`s it** (`encrypt-wallet.mjs:139`
is a bare `copyFileSync`). Under umask `0022` that backup lands **`0644`** —
world-readable — in a directory that is `0700`, so it is protected by the parent
today, but it is one directory-mode change away from being readable, and it is
the single place a seed exists twice.

**Fix (one line):** `chmodSync(backupPath, 0o600)` immediately after the copy —
or better, prompt and delete the backup on successful verification.

**Correction to the brief, recorded here rather than by editing any existing
claim:** the accepted risk as it stands today is *"one plaintext wallet file"*, not
*"a plaintext file plus a plaintext backup"*. The backup is a latent risk created
by the remediation path, not a current one.

---

## 9. Secrets hygiene in logs — clean, with one sharp edge

**No seed or private key can reach a log or an error message.** Verified by
grepping every `console.*`/`throw` in `lib/`, `scripts/`, `bin/` for
`mnemonic|privateKey|passphrase|seed`:

- Every hit is a **generic message** — `'passphrase required for wallet
  encryption'`, `'decryption failed: wrong passphrase or corrupted wallet'`
  (`lib/wallet-encryption.mjs:92,112,174`). None interpolates the value.
- `scripts/create-wallet.mjs:41` **does** print the mnemonic — that is the entire
  purpose of the command, and it prints only on creation.
- No error handler prints an object that could contain key material.
  `send-token.mjs:302` prints `e.stack`, which is a code location, not a secret.

**The sharp edge: `create-wallet.mjs` prints the seed to stdout.** On a desktop
that is fine. Run under a systemd service or with `2>&1 | tee`, it lands in the
journal. **Judgement:** add a `--no-print` flag for scripted creation, and note
that the mnemonic on stdout is also in the terminal scrollback forever.

A second, quieter one: `lib/wallet-encryption.mjs` imports `timingSafeEqual`
(`:31`) and re-exports it in `__test` (`:240`) — but **nothing in the decrypt
path ever uses it**, because AES-256-GCM's auth tag already fails closed. So the
constant-time comparison the module appears to offer is decorative. Not a
vulnerability; a reader should not assume more than is there.

---

## 10. No rate limiting on value-moving operations

Every script is a one-shot process with a single `BCH_CONFIRM=yes` gate. There is
no cooldown, no daily limit, no per-destination cap, and no distinction between
"send 1000 sats" and "send the entire wallet". **Exploitable today: only by
something that already has the seed or shell access**, so the practical exposure
is a *stolen-session* amplifier rather than a new attack — a compromised
Omarchy session can sweep the wallet in one command, and `sweep.mjs` will
happily consolidate every UTXO under 5000 sats into a single output.

This is the same class as `wallet-threat-model.md` §8's multisig recommendation,
stated as a code fact rather than a hardware one. **Judgement:** the cheap version
is an explicit `--all` flag required before any operation that spends more than a
configurable fraction of the balance. **The strong version is multisig or a
separate co-signer process, and that is a design decision, not a patch.**

---

## Ranked summary

| # | Finding | Exploitable today | Value at risk |
|---|---|---|---|
| 5a | `{}` from broadcast reported as success, in 5 scripts | **Yes** (transport condition; documented class, unfixed) | Misreported state; user believes funds moved |
| 3 | Fabricated BRAINROT category id | Yes, cosmetic only | Wrong decimals if it ever matched |
| 4 | No FT amount / category-length bounds | Yes, **fails closed** | Confusing errors, not loss |
| 8 | `encrypt-wallet` writes an un-chmodded plaintext backup | Yes, **on next run** | Full seed, duplicated in cleartext |
| 10 | No rate limiting on value-moving ops | Only with shell/seed access | Whole balance, in one command |
| 7b | Two libauth versions in the tree | Latent | Signing library could change silently |
| 1 | No key zeroisation | **Theoretical** (JS cannot guarantee it) | Heap snapshot while seed is live |
| 5b | Single-node oracle, no quorum | Yes, requires a hostile server | Wrong balances/fees; bounded by preimage |
| 6 | Unbounded derivation, sequential scans | Local only | CPU while seed in memory |

## Sources

[^ct-spec]: CHIP-2022-02 CashTokens, v2.2.2 (final revision 2023-05-20) — "Token Encoding / Token Prefix / Transaction Output JSON Format" sections. <https://cashtokens.org/docs/spec/chip/> — fetched 2026-10-02; states `PREFIX_TOKEN` at codepoint `0xef`, the `prefix_structure` bitfield (`0x80` reserved, `0x40` has-commitment-length, `0x20` has-NFT, `0x10` has-amount), and the fungible amount range "between 1 and 9223372036854775807".
[^nop33]: Ilias Trichopoulos, "You Can't Wipe a Secret in JavaScript, but You Should Try." <https://www.nop33.com/blog/clearing-secrets-from-memory/> — on why JS zeroisation is best-effort: the runtime may copy the buffer and GC timing is not under program control.
[^electrum-authority]: r/BitcoinBeginners, "Electrum Bitcoin Wallet Hacked. 200 BTC Stolen" — maintainer statement of the Electrum server trust model: no authorized servers, cannot forge signatures, can "lie about account balances" and "not relay a valid transaction." <https://www.reddit.com/r/Cryptocurrency/comments/a9yji3/electrum_wallet_hacked_200_btc_stolen_so_far/>
[^beignet]: `coreyphillips/beignet` issue #1010 — "The server-reported UTXO value is used for fee and change while the signature commits to the real amount, so a lying or MITM Electrum server makes the wallet overpay fees." <https://github.com/coreyphillips/beignet/issues/1010>
[^lockfile]: vlt, "Ship JavaScript? Hang onto your lockfile" — the Sept 2025 `chalk`/`debug` maintainer-phishing hijacks and the Shai-Hulud worm spread through semver ranges; a committed lockfile with integrity hashes is the control that stops them. <https://www.vlt.io/blog/hang-onto-your-lockfile>
[^libauth-npm]: `@bitauth/libauth` on npm — "Beyond having no dependencies of its own, Libauth's functional programming approach makes auditing critical code easier." <https://www.npmjs.com/package/@bitauth/libauth>
[^build-pattern]: `bch-crypto-context` skill, `references/bch-bot-build-pattern.md` §1 — "@bitauth/libauth v3.0.0 (^3.0.0). Avoid `v3.1.0-next.x` — pre-release, API drift." Local: `~/.hermes/skills/openclaw-imports/bch-crypto-context/references/`.

---

*Audited 2026-10-02, read-only: `lib/{sign,wallet,router,tokens,network,wallet-encryption,token-registry,hex}.mjs`, `bin/bch-bot`, all non-test `scripts/*.mjs`, plus `git log` on the working tree. No code was modified, no transaction signed or broadcast. Commands and their verbatim output are quoted inline; every file:line reference was read from the live tree at git `d63bda9`.*
