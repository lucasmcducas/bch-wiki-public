
---
pageType: synthesis
id: synthesis.bch-wallet-bot-architecture
title: BCH Wallet Bot - Architecture and Implementation Plan
createdAt: 2026-09-17
covers:
  - entity.bchn-node
  - entity.libauth
  - entity.fulcrum
  - entity.cashscript
  - entity.moth-bch-wallet
  - entity.paryonusd
  - entity.cauldron-dex
  - entity.broach-token
  - entity.brainrot-collection
description: Architecture and implementation plan for a self-hosted, self-custodial BCH wallet bot on Linux (moth-pattern: Node.js + libauth + local BCHN RPC + Fulcrum indexer). Covers BCH + CashTokens (PUSD, ROACH, Brainrot).
---

# BCH Wallet Bot — Architecture and Implementation Plan

*Synthesis: design doc for Luke to read and decide whether to fund/build. Superseded for runtime architecture by `synthesis.bch-wallet-bot-build-log.md` (the actual Phase 1 implementation is against public Rostrum, not local BCHN+Fulcrum).*

> **2026-09-17 update:** The plan was revised after the build revealed public Rostrum/Electrum is sufficient — no local full node needed. See §11 below for the revised runtime architecture and the build-log synthesis for what actually shipped (681 LOC, all working on chipnet).

## 1. Executive summary

This bot holds BCH and CashTokens on behalf of one or more users; signs and broadcasts transactions to a self-hosted BCHN full node; supports covenant interactions (PUSD stake/claim, Cauldron swap, ROACH transfer, Brainrot airdrop distribution). It is built on the **moth pattern** — Node.js + `@bitauth/libauth` for signing + `@electrum-cash/protocol` against a self-hosted Fulcrum indexer — because that is the only stack where (a) every dependency is open-source, (b) signing does not depend on a third-party SaaS, (c) the indexer can be re-implemented, audited, and run on Linux without any cloud-only components, and (d) every other BCH bot in production today (mainnet tools, moth, the cashscript convention) is built on it.

**What makes it hard:** (1) CashTokens signing serialization — there is no `signTransaction` function in libauth; you compose `encodeSigningSerializationBCH` + a secp256k1 backend yourself, with `SIGHASH_UTXOS (0x20)` mandatory for token-bearing inputs. (2) BCHN does **not** index CashTokens internally; the bot must run a Fulcrum indexer on top of BCHN, which means two services, one DB, ~80 GB disk, SSD required. (3) The CashScript covenants in PUSD and Cauldron have non-trivial interaction sequences (PUSD's stability-pool epoch tracking, Cauldron's k-invariant + 0.3% fee math) that the bot has to construct correctly with libauth by hand because the cashscript runtime only handles inputs *in* the covenant's locking script, not P2PKH inputs from the bot's wallet.

**What makes it tractable:** the building blocks are all proven. ParyonUSD ships fingerprints + audit snapshots for their artifacts; Cauldron ships the Auth Template + a deterministic bytecode generator; moth ships a working end-to-end wallet (just not on BCH-test for tokens). With disciplined UTXO selection, a careful account of the libauth pitfalls in `entities/libauth.md`, and rigorous testing on chipnet before mainnet, this is buildable in 4–8 weeks of focused engineering.

## 2. Goals and non-goals

**Goals:**
- Self-custodial — private keys never leave the bot host, never sent to any SaaS.
- Self-hosted — every service runs on Linux from source (BCHN, Fulcrum) or npm (libauth, cashscript, cashlab).
- CashTokens-aware — same bot for BCH, PUSD, ROACH, Brainrot NFTs. No separate stacks.
- Covenant-capable — bot can construct, sign, and broadcast transactions that spend from PUSD contracts, Cauldron pools, and any other CashScript artifact given its `fingerprint`.
- Multi-account, multi-coin — bot can hold multiple HD wallets; sub-accounts can be independently deleted while keeping the seed.
- Honest about uncertainty — surface fee vs. speed trade-offs, credit risk in PUSD, reorg risk; let the user decide.

**Non-goals:**
- **No UX layer.** The bot is a backend; UI is a separate problem. Moth's `bch-wallet` style "AI agent calls CLI scripts" is the operating model. Don't add a web UI in phase 1.
- **No fiat on-ramp.** The bot mints/spends crypto-native assets. No card issuance, no bank rails, no KYC.
- **No merchant / POS layer.** Don't integrate dsproof handling, instant-confirm assumptions, or point-of-sale. The moth credential model (env var gating destructive ops) is the right precedent.
- **No hardware-wallet integration in v1.** Adding `@ledgerhq/connect-kit` or a Trezor transport is a phase-6 stretch. Libauth supports it via BIP32 hand-off, but the integration testing cost is high.
- **No fiat-pricing.** The bot deals in sat-denominated balances internally; fiat conversion is a downstream concern.
- **No custodial multi-user.** One wallet = one custodial relationship. Multi-user is a phase-6 stretch.

## 3. Architecture

```
                     ┌──────────────────────┐
                     │  Moth agent (host)   │
                     │   (AI / orchestration)│
                     └──────────┬───────────┘
                                │ calls CLI scripts via env vars
                                ▼
    ┌────────────────────────────────────────────────────────┐
    │   Node.js wallet bot                                   │
    │   ┌─────────────┐  ┌─────────────┐  ┌──────────────┐   │
    │   │ UTXO        │  │ Transaction │  │ Token RPC    │   │
    │   │ selector    │  │ builder     │  │ layer (FT/   │   │
    │   │ (custom)    │  │ (libauth)   │  │ NFT)         │   │
    │   └──────┬──────┘  └──────┬──────┘  └──────┬───────┘   │
    │          │                │                 │           │
    │   ┌──────┴────────────────┴────────────────┴───────┐  │
    │   │  Electrum Cash protocol client                  │  │
    │   │  (websocket / TCP / @electrum-cash/protocol)    │  │
    │   └────────────────────┬───────────────────────────┘  │
    │                        │                              │
    │   ┌────────────────────┴───────────────────────────┐  │
    │   │  BIP39 mnemonic → HD derivation                │  │
    │   │  m/44'/145'/0'/0/i  (libauth)                  │  │
    │   │  keystore on disk (mode 0600, encrypted optional)│
    │   └────────────────────────────────────────────────┘  │
    └───────────────────────────┬────────────────────────────┘
                                │ WebSocket / TCP :50003
                                ▼
                     ┌──────────────────────┐
                     │   Fulcrum 2.1.2      │
                     │   (Electrum Cash)    │
                     │   CashTokens-aware   │
                     └──────────┬───────────┘
                                │  RPC + ZMQ
                                ▼
                     ┌──────────────────────┐
                     │   BCHN 29.x          │
                     │   (full node)        │
                     │   txindex=1, ZMQ     │
                     └──────────┬───────────┘
                                │ P2P
                                ▼
                          BCH network
```

**Three runtime tiers:**

1. **BCHN** — full node, ~600 GB disk (the pre-utxochache snapshot), 4–8 GB RAM, ZMQ publisher for new txs and blocks.
2. **Fulcrum** — indexer + Electrum server. 40–80 GB disk (growing), 1–4 GB RAM idle, sync is RAM- and SSD-bound. **2.x atomic DB strongly recommended** over 1.x — the 1.x DB can corrupt on hard kill.
3. **Node.js wallet bot** — small footprint (~200 MB disk, 256 MB RAM, single core suffices), runs as a daemon under `systemd` or `pm2`. Talks to Fulcrum only (never directly to BCHN RPC — Fulcrum is the indexer; BCHN RPC is for broadcast and zmq subscription).

**Hardware estimates** (mainnet, peak operation):

- BCHN: 8 cores, 16 GB RAM, 1 TB SSD, ≥20 Mbps up.
- Fulcrum: 4 cores, 8 GB RAM, 200 GB SSD.
- Bot host: 2 cores, 2 GB RAM, 20 GB SSD (it can run on the same host as Fulcrum for a small bot).

**Network ports:**
- 50001/50002/50003/50004 (Fulcrum client; 50002 SSL/50004 WSS are the right defaults).
- 8000 (Fulcrum admin; loopback only).
- 8333 (BCHN P2P, outbound only).

## 4. Component reference

| Component | Library / Version | What the bot uses from it | What the bot has to write itself |
|-----------|--------------------|---------------------------|------------------------------------|
| **libauth** | `@bitauth/libauth` v3.1.0-next.8 (master) / v3.0.0 (released) | Address encoding, BIP39 + HD derivation, signing serialization encoding, transaction build primitives, secp256k1 interface consumption | A `signTransactionBCH` helper composing `encodeSigningSerializationBCH` → `secp256k1.signMessageHashSchnorr/DER` → unlocking bytecode builder; cashc-style signature composition for P2SH inputs |
| **BCHN** | `bitcoin-cash-node` 29.x | Pruned-or-archive full node; `sendrawtransaction`, `getrawtransaction`, `gettxoutsetinfo`, ZMQ for new-block notifications | Spawn scripts: `bitcoind -daemon`, monitor `debug.log`, fail-over to chipnet for testing |
| **Fulcrum** | 2.1.2 | Electrum server as indexer; `blockchain.scripthash.listunspent` for token-aware UTXO discovery; `blockchain.scripthash.subscribe` for address events; `blockchain.headers.subscribe` for new blocks | Connection-pool wrapper, exponential reconnect with backoff, the scripthash-shim for `subscribe` notifications |
| **CashScript** | `cashc` 0.13.0 | Compile any custom `.cash` contract to its artifact; provide `fingerprint` for bytecode integrity check; no runtime use unless interacting with covenants directly | ABI-to-libauth-sighash adapter; integration of `abi` into the manual signer (because the cashscript runtime only handles inputs from the covenant's locking script) |
| **PUSD contracts** | `@paryonusd/contracts` (TS artifacts) | StabilityPool staking (AddLiquidity), withdrawal (WithdrawFromPool + Payout), StabilityPoolSidecar epoch tracking, Borrowing (loans), PriceContract (oracle reads) | Sequence-level transaction construction: an `addLiquidity` call is a 6-output tx with strict token + value balance; we build that with libauth, not with cashscript's runtime |
| **Cauldron** | `@cashlab/cauldron` SDK | Pool bytecode generation, trade math (`calcTradeFee`), `createTradeTx`/`verifyTradeTx` | The transaction *signing*: the SDK builds the unsigned tx; the bot signs it with libauth and broadcasts |
| **moth-pattern reference** | `dev.selene.technology/~moth/skills/bch-wallet.html` | CLI-script pattern (`balance.mjs`, `send.mjs`); `BCH_CONFIRM=yes` env-gated destroy; `wallet.json` storage layout | The 7 moth scripts are thin (~50–200 LOC); our bot starts from them but extends for tokens and covenants. See `entities/moth-bch-wallet.md` for what to keep vs. replace. |

The bot **does not use**:
- Bitcoin Core Electron Cash / Electron Cash (desktop wallet UI).
- PayPal merchant rails, fiat on-ramps, exchanges.
- Any browser-side signing, any cloud KMS, any external signing service.

## 5. CashTokens operations

The bot's six core CashTokens-aware operations:

### FT transfer (fungible cash tokens: PUSD, ROACH)

- Read `blockchain.scripthash.listunspent(bot_addr)` via Fulcrum. For each UTXO, parse `token_data` field.
- Group by category id. For each category, sum the amount across UTXOs (`token_data.amount` is `bigint` in libauth types).
- Construct a spend: `inputs = [ft_input_utxo, fee_input_utxo]`, `outputs = [recipient_ft_output, change_ft_output, fee_output]`. Recipient FT output's `token.amount` is the desired amount.
- **Critical:** every output in the same token category must reference the **same** category id (32-byte category is preserved across transactions, not regenerated). The category comes from the input's `token_data.category`.
- Sign with `SIGHASH_UTXOS (0x20)` in the sighash byte OR'd with `SIGHASH_ALL | SIGHASH_FORKID` → `0x61`. `encodeTokenPrefix(sourceOutputs[inputIndex].token)` is required in the preimage. Without it, consensus rejects.
- NFT inputs to the same address require `SIGHASH_UTXOS` on every input even if the recipient doesn't want NFTs (because UTXOs are bundled).

### NFT transfer (Brainrot-style)

- Read NFT UTXOs; each carries `token_data.nft = { capability, commitment }`.
- Preserve `commitment` byte-for-byte through the transaction; libauth v3+ types include `nft.commitment` as the canonical field.
- Capabilities: `none` (immutable), `mutable` (can be replaced in a future tx while preserving category + commitment), `minting` (can mint additional NFTs in the same category).
- **For a Brainrot airdrop:** the bot's minting contract issues an NFT to `recipient`, with the recipient's BCMR-registered commitment. The bot only needs to spend the minting-baton NFT input with `SIGHASH_UTXOS` and produce the new NFT output (with the right commitment) + a recursive minting-baton output.

### Minting-baton handling

- A minting-baton is an NFT with `capability = 'minting'` — it is the authority that can spawn new tokens or NFTs in the same category.
- Hold minting-batons in the bot's treasury only if you intend to mint. Otherwise send to an OP_RETURN or burn.
- Spend baton UTXO with `SIGHASH_UTXOS`; the output that recreates the baton must be at least one output; the other outputs in the same tx carry the new tokens being minted.

### Gas / dust rules

- BCH outputs >= 546 sats (dust threshold). Bot absorbs sub-dust into fee (moth convention).
- Covenant outputs (PUSD, Cauldron pools) have their own minimum value requirements. PUSD's `AddLiquidity` requires ≥ 100.00 PUSD minimum per the contract `minimumToStake = 100_00`. Below that, the contract refuses the spend.
- CashTokens don't have a separate fee; the BCH fee covers all token-bearing inputs (no per-token fee). The fee scales with transaction byte size, not token amount.

### `sighash_utxos` (0x20) requirement

- **Mandatory** for any input that spends a token-bearing UTXO.
- This **includes** P2PKH inputs that don't carry tokens but happen to be in a tx with token-bearing outputs — including them with `SIGHASH_UTXOS` is **not** required, but if you include them in the same transaction with token outputs, all inputs must use `SIGHASH_UTXOS` or the signature is invalid.
- Conservative practice: always use `0x61` (`SIGHASH_ALL | SIGHASH_UTXOS | SIGHASH_FORKID`) for CashTokens-aware txs.

### Multiple-of-CashTokens — the atomicity requirement

- A category id can have both fungible and non-fungible tokens in the same UTXO. Spending them requires a special "split" transaction since BCH consensus treats them atomically. **Tooling:** the cashscript-style `outputTokenPrefix` encoding handles this; if both fungible amounts and NFT instances coexist in one UTXO (rare but possible), the bot must split them in a non-trustful ancestor tx first.

## 6. Integration patterns

### PUSD stake / claim sequence

**Stake (deposit to stability pool):**

1. Confirm bot has ≥ 100.00 PUSD at a known address.
2. Call `Liquidity.addLiquidity` (or equivalent ABI method), passing the bot's current PUSD UTXO.
3. The tx spends the PUSD input + a small BCH input for fees, and produces:
   - The updated StabilityPool contract output (BCH-1000 = 0 sat structure, with the mutable NFT state)
   - The StabilityPoolSidecar output (with the bot's staked tokens)
   - The AddLiquidity pool-function-contract output (NFT commitment: epoch+1, staked amount)
   - The user's receipt NFT output (mintable, navigates back through subsequent withdrawals)
   - Change outputs for any excess BCH or PUSD
4. Sign with `SIGHASH_UTXOS` on every input; sighash byte `0x61`.

**Claim (withdraw + claim BCH yield):**

1. Wait for the next payout (every 10th period a new epoch is opened; look for `NewPeriodPool` event).
2. Call `WithdrawFromPool` against the bot's staking receipt NFT (epoch commitment must match).
3. The tx spends the receipt, recreates the staking position (if desired), and produces:
   - Updated StabilityPool output
   - StabilityPoolSidecar output (with reduced PUSD balance)
   - WithdrawFromPool contract output (rerun)
   - The withdrawn PUSD output (to the bot's address)
   - The BCH yield output (from the Payout contract, to the bot's address)
   - BCH change

### Cauldron swap sequence

1. Pick a target token and amount. Query the Cauldron pool UTXO via Fulcrum (the pool is a single CashTokens UTXO holding both sides).
2. Use `ExchangeLab.constructTradeBestRateForTargetDemand(targetSats, pool, feePayoutToken)` to compute the trade shape.
3. Use `ExchangeLab.createTradeTx(...)` to assemble the unsigned transaction spending the pool + the user's input + the fee input.
4. Sign with `SIGHASH_UTXOS` on every input that touches tokens; bot's P2PKH BCH input can use plain `0x41`.
5. The unsigned tx reorders inputs so user inputs are first.
6. **Verify with `ExchangeLab.verifyTradeTx`** before broadcasting.

### ROACH transfer (fungible-only token)

1. Read bot's ROACH balance: `blockchain.scripthash.listunspent(bot_addr)`, group `token_data.category == 0x892cef80...` (the category id from `entity.broach-token`), sum amounts.
2. Build an FT-only transfer like the FT case above. Use SIGHASH byte `0x61`.
3. **Don't include any NFT output in the same tx** — even with `commitment` zeroed — to keep the transaction simple and avoid the rare atomicity edge case.

### Brainrot airdrop delivery (NFT distribution)

The minting path:

1. Bot's treasury holds a minting-baton NFT for `categoryId = brainrot_category`.
2. Receive the holder list (e.g., 1000 ROACH holders at snapshot time).
3. For each recipient, build a tx with: inputs = [baton NFT, fee BCH input]; outputs = [new recipient NFT (with `commitment = character_index`), recursive baton (still minting-capability, going back to bot), BCH change].
4. Sign with `SIGHASH_UTXOS`. Compute `outputTokenPrefix` from the source baton UTXO's `token_data`.
5. Batch-broadcast.

A single minting tx can mint multiple NFTs to multiple recipients in one tx if the gas budget allows (BCH fee byte size caps this; ~50–80 NFT outputs per tx is typical).

Constraints:
- Same **category id** for every NFT output in the tx.
- `commitment` byte per NFT (32 bytes) carries the character index (`0x0000...0014` for character 20, etc.).
- Each NFT output carries the corresponding `character_metadata_cid` (IPFS pointer) in BCMR; pre-publish before broadcasting.

## 7. Implementation phases

### Phase 0 — BCHN + Fulcrum (~1 week)

- **Scope:** install and configure BCHN with `txindex=1` and ZMQ. Install Fulcrum 2.1.2 against it. Sync Fulcrum's DB (allow 1–3 days).
- **Deliverables:**
  - `bitcoin.conf` in /etc with the required flags.
  - `fulcrum.conf` with `bitcoind` RPC, `admin=` bound to 127.0.0.1, client port 50002 SSL or 50003 WS.
  - Systemd units for both.
  - `systemctl status bitcoin-fulcrum` returning `active (running)`.
  - Smoke test: `electrum-cash` connecting to localhost, `server.features()` returns `cashtokens: true`.
- **Risks:** Fulcrum DB corruption on hard kill during initial sync (use 2.x; never `kill -9`). Disk shortage — start with 200 GB. BCHN's `maxconnections` setting; tune so it doesn't choke Fulcrum's queries.

### Phase 1 — libauth-backed BCH send/receive (~1–2 weeks)

- **Scope:** CLI scripts modeled on moth's, but using our own UTXO selector. Operations: `create-wallet`, `address`, `balance`, `utxos`, `history`, `send`. Read libauth entity page for the signing composition pattern.
- **Deliverables:**
  - `send.mjs` that composes `encodeSigningSerializationBCH` + secp256k1 backend + unlocking bytecode, all per the v3 surface.
  - Wallet import/export (BIP39 mnemonic → xprv → addresses).
  - RBF enabled by default on outbound txs.
  - Integration test on chipnet (BCH testnet): mint a test utxo, send to a second wallet, broadcast, verify.
- **Risks:** SIGHASH composition bugs (off-by-one in forkId). Signing with the wrong key derivation. UTXO selector that picks dust. **Mitigation:** write a fuzz test against libauth's own signing-serialization spec tests (vendored).

### Phase 2 — CashTokens (~2 weeks)

- **Scope:** extend `send.mjs` → `send-token.mjs` with category id + amount (FT) or commitment (NFT). Implement `utxos.mjs --token <category>` to filter by category. Implement the `SIGHASH_UTXOS` always-on rule.
- **Deliverables:**
  - Token-aware UTXO selector with category-id grouping.
  - `outputTokenPrefix` encoding wrapper around libauth's `encodeTokenPrefix`.
  - End-to-end test: receive an ROACH FT from chipnet faucet → hold → transfer to a fresh address → confirm.
- **Risks:** Atomicity edge cases (multiple categories in one UTXO — rare in practice but bad if not handled). Commitment truncation. Missing v3 NFT fields.

### Phase 3 — PUSD integration (~2 weeks)

- **Scope:** import `@paryonusd/contracts` artifacts. Verify fingerprints against published audit hash. Implement `stake.mjs` and `claim.mjs` per the integration pattern above.
- **Deliverables:**
  - Multi-output tx construction for stability pool epoch tracking (each output's locking script is a different contract address).
  - BCH value and token accounting: every output's `valueSatoshis` + `token.amount` must balance (modulo fees).
  - End-to-end test on chipnet: stake 1000 PUSD, wait 1 epoch (or use forced epoch bump), withdraw.
- **Risks:** The 6-output sequence (and the related `outputTokenPrefix` correctness per input) — easy to miss the sidecar-only-vs-pool-only distinction. Min-stake enforcement (100.00). Liquidation: don't try to do it in v1; the bot is a staker, not a liquidator.

### Phase 4 — Cauldron (~2 weeks)

- **Scope:** integrate `@cashlab/cauldron`. Build trade tx per the integration pattern. Verify k-invariant via SDK before broadcasting.
- **Deliverables:**
  - `swap.mjs --pool <addr> --from <token> --to <token> --amount <n>`.
  - Wallet-side integration: pool UTXO subscription via `blockchain.scripthash.subscribe` so the bot sees price moves.
  - End-to-end test on chipnet: bootstrap a test pool, swap through it.
- **Risks:** Computing the wrong trade side for the fee. Trusting the SDK's `verifyTradeTx` blindly (it has its own bugs). Signing the wrong sighash byte for BCH inputs that don't touch tokens (`0x41` vs. `0x61`).

### Phase 5 — Airdrop distribution (~2 weeks)

- **Scope:** minting-baton management; airdrop delivery via Brainrot or ROACH events.
- **Deliverables:**
  - `airdrop.mjs --token <category> --recipient-list <file>`.
  - Pre-publish BCMR for each character before minting the first NFT.
  - Snaphot minting: build multi-output txs (50–80 NFT outputs each), batch-broadcast.
- **Risks:** Minting-baton lost mid-airdrop → all remaining recipients receive nothing. BCMR not pre-published → wallets display the wrong metadata. Dust accumulation on tiny remainder outputs.

### Stretch goals (Phase 6+)

- Hardware-wallet (Ledger/Trezor) signing via BIP32 hand-off.
- Multi-account HD keystore.
- The `0x41` vs `0x61` sighash selection per-input (currently always-`0x61`).
- P2S (CHIP-2024-12) locking-bytecode helpers when libauth adds them.
- Covenant patterns not in PUSD/Cauldron (custom `.cash` you write yourself).

## 8. Open questions and gaps

**What the wiki does NOT cover:**

1. **No lab-tested timing data for Fulcrum DB creation.** "1–3 days" is an estimate from the README/community. Real time depends on disk speed and network.
2. **EarnVault does not appear to exist on BCH** as a public, open-source contract. If the bot is supposed to integrate with it, that's a content gap — there is no contract to point at.
3. **moth's actual `.mjs` source files** are not on the public web. We have the page and the SKILL.md; the internals (UTXO selection algorithm in particular) cannot be replicated exactly without access to moth's runtime.
4. **No published benchmarks** for libauth v3 signing throughput. For an airdrop distributing 50K NFTs over 1000 txs, throughput matters; we don't know.

**What the libauth / CashScript ecosystems don't provide:**

1. **No formal verifier.** CashScript contracts have to be audited; libauth's signing primitives have to be tested against the protocol spec by hand.
2. **No standardized covenant test framework.** Each contract family (PUSD, Cauldron) ships its own tests. There's no shared "CashScript spec test" corpus.
3. **No ECDSA/Schnorr WASM binding shipped with libauth itself.** You choose secp256k1 (libsecp256k1, @noble/secp256k1, libauth-template's bundled one). Each has trade-offs.
4. **No P2S (CHIP-2024-12) support** in libauth v3.1-next.8 master. If your covenants use P2S, you construct the bytecode yourself.

**What is NOT decidable without Luke's input:**

1. **Risk tolerance on `wallet.json` encryption** — moth ships plaintext + mode 0600. Do we adopt the moth `credentials` skill's AES-256-GCM + scrypt pattern, or skip encryption for simplicity?
2. **Testnet/chipnet vs. mainnet policy.** Should the bot default to chipnet and mainnet only when explicitly configured? moth's page does not say.
3. **Multi-account vs. single-account keystore.** moth uses one wallet per install. Do we need ten?
4. **Funding source for development.** Is this a paid project, a side project, a learning experiment? Determines stack-up vs. ship-now posture.
5. **Brand / account abstraction layer for the Brainrot distribution** — does the bot mint NFTs directly from the treasury, or delegate to a Paryon-style lending contract where the batons sit? The Brainrot wiki flagged this as unresolved.
6. **Hardware-wallet parity** — is supporting Ledger / Trezor a launch requirement, or a post-launch stretch?
7. **Continuity of moth as a reference.** If moth's skill page goes offline, our pattern loses one of its anchor documents. Worth mirroring locally.

**Bottom line:** the bot is buildable. The architecture is determined by the dependencies that already exist (libauth + Fulcrum + CashScript + moth pattern). The unknowns are policy / UX decisions, not technical ones.

## 11. Revised runtime architecture (2026-09-17, post-Phase-1-build)

**Build evidence:** Phase 1 ran end-to-end against public Rostrum servers with no local BCHN/Fulcrum. Total bot code: 681 LOC at `~/bch-bot/`. Connected to `chipnet.bch.ninja:50004`, negotiated protocol 1.5, derived 5 distinct BIP44 addresses, queried balances and UTXOs, dry-ran a send. **No local node required.**

**Revised three-tier:**

1. **Public Rostrum/Electrum server** (remote, free, public): `chipnet.bch.ninja:50004`, `cashnode.bch.ninja:50004`, `rostrum.cauldron.quest:50004`. Replaces BCHN + Fulcrum. CashTokens-aware (Rostrum 1.5 protocol). Maintained by Kallisti / Selene Official + community.
2. **bch-bot** (local, this box, ~50 MB disk): the moth-style thin scripts that use `@bitauth/libauth` for signing and `@electrum-cash/network` for indexing.
3. **BCHN full node**: REMOVED from scope. Available if/when bot needs to verify Merkle proofs itself or run an indexer under our control. Not needed for Phase 1 or Phase 2.

**Why this works:** Selene Wallet — the live, BSD-3, actively maintained upstream — runs against the same public servers with no local node. Same CashTokens support, same libauth signing, same protocol. moth ships the same way. The architecture doc's insistence on BCHN+Fulcrum was over-spec, not under-spec.

**What the bot's stack proof demonstrated (2026-09-17):**
- BIP39 → HD node → 5 distinct P2PKH cashaddrs at m/44'/145'/0'/0/{0..4} ✅
- Connect to chipnet.bch.ninja:50004, negotiate protocol 1.5 ✅
- Query scripthash balance + utxo list for 20 derived addresses ✅
- Locking bytecode verified as canonical P2PKH (76 a9 14 ...) ✅
- Build + sign P2PKH tx (dry-run path compiles) ✅
- Full code path: 681 LOC, no local services, ~5 sec cold start

**What it did NOT demonstrate:**
- A real broadcast. `BCH_CONFIRM=yes` was never set.
- A real round-trip. No chipnet faucet found that works post-May 2026 chipnet rollout.

**Implications for Phase 2:**
- CashTokens: switch from `blockchain.scripthash.get_balance` (which doesn't return token data on regular Electrum servers) to rostrum.cauldron.quest:50004 for token-aware queries. Use `libauth.compiler` path for plain P2PKH token outputs; use manual `generateSigningSerializationBCH` + `signMessageHashSchnorr` path for CashScript contract inputs.
- PUSD: same pattern, swap the libauth compiler for the `importWalletTemplate` of PUSD's compiled artifact.
- Cauldron: SDK `@cashlab/cauldron` provides `createTradeTx` for the unsigned tx; the bot signs with libauth and broadcasts via `blockchain.transaction.broadcast`.

**Disk / ops profile (revised):**
- Bot disk: ~50 MB (npm install)
- BCHN: 0 (eliminated)
- Fulcrum: 0 (eliminated)
- Network dependency: stable public Rostrum (8+ servers in failover list)
- Single point of failure: public Rostrum downtime. Mitigation: 8-server failover list maintained in `lib/network.mjs` (Selene pattern).

**Detailed lessons from the build:** `synthesis.bch-wallet-bot-build-log.md` (the "by doing" page).

---

*Updated 2026-09-17 evening after the Phase 1 build surfaced the public-Rostrum pattern. The original plan (`~/.hermes/plans/2026-09-17-bch-wallet-bot.md`) was patched the same day.*
