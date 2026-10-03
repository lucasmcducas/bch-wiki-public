---
pageType: reference
id: reference.security.cashtokens
description: CashTokens-specific attack surface — commitment formats, minting/smelting, covenant introspection, indexer trust, replay risks, and known vulnerability classes.
sourceUrl: https://cashtokens.org/docs/spec/chip/
---

# CashTokens Security Reference

*Lens: CashTokens-specific attack surface introduced by CHIP-2021-05 and follow-on CHIPs. Complements `reference.security.utxo-and-mempool` (sender-side dust/fee/mempool) and `reference.security.bch-zero-conf` (acceptor-side 0-conf). This page is the canonical place for the FT/NFT primitives' security model. Word count: ~2,400. Authored 2026-09-19.*

## 1. Commitment formats and the category/commitment distinction

CashTokens extends every BCH transaction output with up to **four** new fields: a 32-byte `category`, an NFT `capability` (`none`/`mutable`/`minting`), an NFT `commitment` (0–40 bytes), and a fungible `amount` (1 to 2^63−1 = 9 223 372 036 854 775 807).[^chip-spec]

Two concepts are routinely conflated and must be kept separate:

- **`tokenCategory` (32 bytes)** — the **identity** of the token. It is the `txid` of the genesis transaction that created the category. Every input in the genesis tx must spend output index 0 of its parent, and that parent's `txid` *becomes* the category id. This is consensus-enforced.[^chip-spec]
- **`nftCommitment` (0–40 bytes)** — an **arbitrary message** carried inside a specific NFT. It is *not* unique per token; multiple NFTs of the same category can carry the same commitment unless a covenant forbids it.[^chip-spec]

Libauth models this with `{ category: Uint8Array(32), amount: bigint, nft?: { capability, commitment } }`; `bch-bot/lib/tokens.mjs:30` mirrors this shape via `utxoToTokenPrefix`.[^bot-tokens]

Why the distinction matters for security: the **category** is what gives the token its on-chain identity and is what FT amounts sum against. The **commitment** is just bytes that anyone in the category's covenant can rewrite (with `mutable`/`minting` capability) — it has *no* consensus-level uniqueness, *no* sequence number, and *no* cryptographic binding to the token amount. Treating it like an NFT id is a category-confusion bug (see §6).

### Byte layout (output encoding)

The token prefix is encoded *before* index 0 of the locking bytecode (a backward-compatibility hack so old decoders still work). The CompactSize length prefix is widened to cover `token_prefix + locking_bytecode`. The prefix itself is, in order:

```
[category: 32 bytes]  [amount: varint, present iff FT amount > 0]
  [capability: 1 byte, present iff NFT present; 0=none, 1=mutable, 2=minting]
  [commitment_length: varint] [commitment: 0..40 bytes]
```

The capability byte is **omitted** when no NFT is in the output; the amount varint is **omitted** when amount is zero.[^chip-spec]

### What this means for wallet code

A wallet building a token-bearing output must:

1. Carry the 32-byte category bytes verbatim — never derive from the token name, ticker, or `token_data.amount`.
2. Use the **minimal** big-endian encoding for the amount varint (253+ uses the `fd xx xx` prefix; 9223372036854775807 uses the `ff ff...7f` 9-byte prefix). The full range up to 2^63−1 is allowed, but wallet UIs that use `Number` (not `bigint`) silently corrupt amounts above 2^53.[^chip-spec]
3. Strip the NFT block entirely when sending an FT-only output — adding an empty commitment is wasteful and changes the dust calculation.
4. Pass the **input token prefix** into the signing serialization (libauth: `unlockingBytecode.token`). Skipping this makes the preimage the signer saw differ from the preimage the network verifies, producing `mandatory-script-verify-flag-failed` at consensus.[^bot-sign][^utxo-mempool]

`bch-bot/lib/tokens.mjs:30–43` does this correctly: it reads the input UTXO's `token_data` from Rostrum and translates to libauth's `{category, amount, nft}` shape; the sign layer consumes that shape verbatim.

## 2. Minting and smelting

The CHIP introduces two categories of operations that only contracts (and humans who hold the right NFT) can perform. Both are **consensus-enforced**.

### Minting (creating NFTs)

There are three NFT capability levels:

| Capability | Spending an input with this NFT lets the tx… |
|---|---|
| `none` (immutable) | Create zero new NFTs of this category. Commitment is preserved as the input's commitment. |
| `mutable` | Create **one** new NFT of this category, with any commitment, optionally as `mutable`. |
| `minting` | Create **any number** of new NFTs of this category, each with any commitment, optionally as `mutable` or `minting`. |

A single transaction with `k` minting-NFT inputs can therefore produce up to `k * (unlimited)` NFTs.[^chip-spec]

The **genesis transaction** is itself a form of minting: any input with `outpointIndex == 0` (i.e. spends the 0th output of its parent) is automatically considered a minting-NFT input *for the parent's txid-as-category*. This is how a category is created: spend any 0th output in a tx that produces a token-bearing output with that category id.[^chip-spec]

### Smelting (destroying NFTs implicitly)

This is the **smelting** behavior, and it is the most surprising part of the validation algorithm. NFTs can be **destroyed by omission** — if an output's NFT does not appear in some output of the spending transaction, the NFT is gone. The CHIP uses the word "destroyed" and the algorithm tracks `Available_Immutable_Tokens`, `Available_Mutable_Tokens_By_Category`, and `Available_Minting_Categories` to enforce that *every* immutable NFT in the inputs has a matching output NFT with the **same category AND same commitment**.[^chip-spec]

The downstream rules from the validation algorithm:[^chip-spec]

- For each output NFT with category C and capability `mutable`: C must be in `Available_Minting_Categories` (consume a "mint slot"), **or** the count of output mutable NFTs of C must be ≤ the count of input mutable NFTs of C (consume mutable slots).
- For each output NFT with category C and capability `none`: C must be in `Available_Minting_Categories` (consume a mint slot), **or** there must be a matching input NFT with same category AND commitment (one-for-one pairing), **or** there must be a spare mutable NFT of C available (mutable gets downgraded to immutable in the output).

This last rule — "mutable NFT in input, immutable NFT in output is OK if commitments match" — is a downgrade path and is the foundation of covenant-receipt patterns like PUSD's stability pool.

### Why minting is a covenant primitive

A `minting`-capability NFT is the **only** thing that can produce new NFTs of a category. By issuing exactly one such NFT (per category) into a covenant's locking bytecode, the covenant becomes the sole authority on issuance: every mint must spend the covenant, the covenant enforces the issuance policy, and the minting NFT is recreated at the same index in the output. This is exactly how PUSD's `Borrowing` contract works, how Cauldron uses an `Authhead` NFT to track pool authority, and how CashScript `Mecenas`/`Cashninjas`-style contracts track their own state.[^chip-spec][^cashscript-covenants][^pusd]

## 3. Covenant introspection — what an attacker can read

CashTokens added **six** token-introspection opcodes (0xce–0xd3) on top of the 0xc0–0xcd introspection opcodes already present from the May 2022 native-introspection upgrade:[^chip-spec]

| Opcode | Codepoint | Stack pop | Stack push |
|---|---|---|---|
| `OP_UTXOTOKENCATEGORY` | `0xce` | input index | category, **or** category‖capability-byte (1=mutable, 2=minting) |
| `OP_UTXOTOKENCOMMITMENT` | `0xcf` | input index | NFT commitment bytes (or 0 if absent/empty) |
| `OP_UTXOTOKENAMOUNT` | `0xd0` | input index | FT amount (or 0) |
| `OP_OUTPUTTOKENCATEGORY` | `0xd1` | output index | same shape as `OP_UTXOTOKENCATEGORY` |
| `OP_OUTPUTTOKENCOMMITMENT` | `0xd2` | output index | commitment or 0 |
| `OP_OUTPUTTOKENAMOUNT` | `0xd3` | output index | FT amount or 0 |

Combined with the four general introspection opcodes (`OP_INPUTINDEX/0xc0`, `OP_UTXOVALUE/0xc1`, `OP_UTXOBYTECODE/0xc2`, `OP_OUTPOINTBYTECODE/0xc3`, plus the sighash-all-utxos variant `OP_INPUTBYTECODE/0xc4` and `OP_OUTPUTBYTECODE/0xc5`), a covenant can:

- Verify *which* input is currently being evaluated (`OP_INPUTINDEX` → `this.activeInputIndex` in CashScript).
- Read the FT amount and NFT commitment of any input or output by index.
- Compare input and output token categories byte-for-byte.
- Compute `tokenAmountDelta = sum(out) - sum(in)` for a category, or `commitmentDelta = bytes(out_nft[i]) - bytes(in_nft[j])`.

This is what makes the Streaming Mecenas example in the CashScript docs work — it stores local state in the NFT commitment of the contract's own UTXO and reads it back with `OP_UTXOTOKENCOMMITMENT`/`OP_OUTPUTTOKENCOMMITMENT`.[^cashscript-covenants]

### P2SH-32 — what it limits

P2SH-32 (pay-to-scripthash-32, activated May 2023 with CashTokens) replaces P2SH's 20-byte RIPEMD-160 hash with a 32-byte SHA-256 hash. The motivation was collision resistance: classic P2SH is vulnerable to a 2^80 collision attack, which is feasible for state-level attackers even at 2^82 with trivial memory.[^p2sh32]

For CashTokens security specifically, P2SH-32 means:

- A covenant's address is bound to a 32-byte hash, and the chance of a colliding covenant at the same address is cryptographically negligible.
- But the **redeem script** still executes in the VM, and **all the introspection opcodes above still see into the spending tx**. P2SH-32 does not reduce the introspection surface — it only reduces address collision risk. A covenant author can still write a contract that incorrectly trusts its input amounts, mis-compares commitment bytes, or accepts the wrong category. The 32-byte hash is a defense against address squatting, not against covenant logic bugs.

## 4. Token indexer trust

A wallet does not see FT/NFT balances by parsing every block — it queries an **indexer** (Rostrum/ElectrsCash/Fulcrum/full-node RPC). For BCH CashTokens, the canonical indexers are:[^utxo-mempool]

- **Rostrum** (`rostrum.cauldron.quest:50004` for Cauldron; `cashnode.bch.ninja:50004` for general). Speaks Electrum protocol 1.4.3+ for CashTokens-aware methods, 1.5+ for full CashAddress support.
- **ElectrsCash** (Bitcoin Unlimited fork of electrs). Rust implementation; indexes token-bearing UTXOs and exposes them via `blockchain.scripthash.listunspent` with a `token_data` field.[^electrs-cash]
- **BCHN full-node RPC** (`getrawtransaction` + `gettxoutsetinfo`) — slow but trustless.

### Trust assumptions

A malicious indexer can:

1. **Lie about a balance.** Return a UTXO that doesn't exist, hide a UTXO that does, or report a different `token_amount`. The wallet sees the lie as ground truth and may sign a tx that over-spends or under-sends.
2. **Lie about token metadata.** Claim an FT is a different `decimals` value than it actually is, or attach a fake NFT to a plain BCH UTXO. Wallets that show "your NFT gallery" by parsing `token_data.nft.commitment` from the indexer are at the indexer's mercy.
3. **Lie about confirmations.** Tell the wallet that an unconfirmed tx is confirmed (or vice versa). DSProof subscribers can detect double-spends but not confirmation lies.
4. **Lie about fees and mempool inclusion.** Standard SPV attack: withhold a tx so it never reaches miners, or show a fake "already confirmed" response.[^electrum-trust]

The Spark research summary is exact: *"A malicious server can withhold transactions (making a balance appear lower than it is) or serve incorrect fee estimates, but it cannot forge transactions or steal funds."*[^electrum-trust] For token data, that last clause is mostly still true — but a malicious indexer **can** mislead the wallet into signing a tx that destroys the user's tokens, which is functionally equivalent.

### Mitigations the bch-bot uses

- **Two-server failover** (`lib/network.mjs`): tries `cashnode.bch.ninja` first, then 1-2 fallbacks. Catches outright outages; does not catch consistent lies across all servers.
- **The 1000-sat margin in `selectInputsForTokenSend` (`lib/tokens.mjs:172`)** is partly a defense against indexer miscounts — if the indexer reports a UTXO at 1000 sats that's actually 800 sats, the tx still confirms and pays the difference as extra fee, but doesn't get rejected.
- **No tx-signing based on untrusted metadata alone.** All signing operations in the bot pass the token prefix through libauth, which independently serializes it. The preimage is recomputed locally; the indexer's word is taken only on which UTXOs to spend.

### Mitigations the bot does *not* yet use (gaps)

- **No SPV proof verification.** The bot doesn't fetch block headers and Merkle-proof an inclusion claim. For high-value token operations, the operator should run their own full node and trust `getrawtransaction` directly.
- **No DSProof subscription for token UTXOs.** The bot subscribes to nothing; if the indexer withholds a conflicting tx (or if a real double-spend occurs), the bot doesn't know.[^utxo-mempool]
- **The bot's `cauldron.mjs:101` calls `cauldron.contract.subscribe`** — a Cauldron-specific method. A malicious Rostrum at that endpoint can return fabricated pools with fake `sats` and `token_amount`, and the bot's `getTokenPrice` will compute against the fake data.

## 5. Replay and cross-category confusion

### Cross-category replay

Because CashTokens uses 32-byte category ids, two tokens **cannot** collide on category. A tx that says "transfer 100 PUSD" with category `2469acc5…` cannot be replayed against "100 ROACH" with category `892cef80…` — the network rejects at the input/output sum check (the input 100 ROACH doesn't equal the output 100 PUSD per category).

**But**: the *signature* on a tx commits to the inputs, outputs, and the sighash flag. If a wallet signs a tx where the token category in the sighash matches the input UTXO, then any replay of that signed serialization must use the same UTXO — the inputs are committed by outpoint. So classical signature-replay is structurally blocked.

The **remaining** replay class is structural:

- A tx that burns FT (no FT output for the category, but other outputs exist) is **not** replayable; the FT inputs would have no matching outputs.
- A tx that spends a `minting` NFT and produces new NFTs of the same category: the new NFTs are tied to the same category as the spent NFT, so the network won't accept them as belonging to a different category.

### Accidental FT+NFT commitment interactions

The riskier surface is **implicit category sharing**: every output carries at most one category, and any amount of FT plus at most one NFT must all be of the same category. The CHIP says: *"all tokens in an output must share the same token category."*[^chip-spec]

What this means in practice: if you receive an FT of category C and an NFT of category D in two separate UTXOs, you cannot merge them into one output. Most wallets don't try to, but **autcoin-merge / sweep tools** that consolidate UTXOs can silently fail or create invalid txs if they don't understand the category boundary. `bch-bot/scripts/sweep.mjs` is BCH-only — it explicitly filters out token-bearing UTXOs from sweep candidates, which is the right behavior.[^utxo-mempool]

A more subtle case: a covenant that requires "the output at index `i` has commitment bytes X" without also constraining the **category** can be tricked into accepting a different category's NFT that happens to have commitment X. The fix is to require both `OP_OUTPUTTOKENCATEGORY(i) == expectedCategory` AND `OP_OUTPUTTOKENCOMMITMENT(i) == expectedCommitment`. PUSD's `AddLiquidity.cash` requires both.

### How the commitment scheme prevents the worst classes

The commitment is **not** an NFT id (commitments can repeat), but it **is** a per-NFT byte string that's either preserved (immutable), one-to-one matched (mutable), or arbitrary (minting). The covenant pattern is:

- The **minting contract** receives a mint request with a *new commitment value* and writes it into the output's commitment. The new commitment is uniquely bound to the input request (e.g. hashed against the recipient pubkey + a sequence number).
- A separate **indexer** (or off-chain registry) maintains the canonical mapping from commitment → metadata.

Brainrot Battles uses this pattern: the per-NFT commitment stores the character index (0–19) so the card game can read stats from a wallet without an off-chain DB.[^brainrot]

## 6. Known vulnerability classes

These are the categories of bugs the author has seen or expects to see, ordered roughly by likelihood.

### 6.1 Incorrect commitment generation (off-by-one, endianness, padding)

CashScript's `bytes(int)` returns the *minimal* big-endian encoding — `bytes(10000) = 0x2710`, not `0x00002710`. The `bch-bot/lib/pusd.mjs:90` `buildReceiptCommitment` correctly uses 4-byte big-endian for `nextEpoch` (the padded portion) and minimal big-endian for `addedTokenAmount` (the variable portion), and concatenates them — matching the contract's `toPaddedBytes(nextEpoch, 4) + bytes(addedTokenAmount)` shape.[^bot-pusd]

A common bug: **padding the amount to 4 bytes when the contract expects minimal**. This produces a different commitment, the covenant's `OP_HASH256` check fails, and the covenant locks up. The fix is to read the contract's encoding spec line-by-line and verify the bot's hex output.

### 6.2 Smelting edge cases (last-output burns)

A tx that spends a category's only minting NFT but produces no output NFT for that category **destroys** the minting NFT — and with it, the entire category's ability to mint further NFTs. (It does not destroy existing FTs, but the FTs become unmintable and any future NFTs of the category become unspendable.)

This is a real footgun: a covenant migration that fails to recreate the minting NFT in the output can permanently brick the category. `CashScript`'s `withdraw_*` patterns and ParyonUSD's `Borrowing.cash` all explicitly require `tx.outputs[0].lockingBytecode == tx.inputs[0].lockingBytecode` AND preserve the NFT — but a contract author who forgets the NFT recreation clause (only requires the locking bytecode match) burns the minting capability.

The CHIP's validation algorithm catches some forms (consensus rule: every input minting token must have an output minting token of the same category), but it does **not** require commitment preservation for minting NFTs — only for immutable ones.[^chip-spec]

### 6.3 FT amount overflow (uint64 / 2^63−1)

The maximum FT amount per category is **2^63 − 1 = 9 223 372 036 854 775 807** (the max VM number). Above this, the amount varint cannot be encoded, so a tx trying to mint more is consensus-invalid.[^chip-spec]

However, code paths that do `BigInt(amount) + BigInt(amount)` in a loop or `parseFloat(amount)` in JS can:

- **Wrap around on `Number`**: anything above 2^53 silently loses precision. Libauth always uses `bigint`, and the bot follows suit (`lib/tokens.mjs:64`); but a wallet that imports amounts as `Number` from an Electrum server can corrupt them.
- **Overflow on intermediate sums**: spending 5 UTXOs of 9 223 372 036 854 775 000 each and adding in JS with `BigInt` does not overflow (BigInt is unbounded), but the *output sum* must still be ≤ the *input sum*, and any single output's amount varint must encode in ≤ 9 bytes. The validations check the second implicitly.

### 6.4 Category confusion attacks

A covenant that verifies `OP_UTXOTOKENCATEGORY(0) == expectedCategory` but does **not** verify the **capability** byte can be tricked by an attacker providing an immutable or mutable NFT where a minting NFT was required. The result: the covenant reads category bytes that look right but the NFT isn't authorized to mint.

The CHIP's introspection opcode returns category bytes *concatenated with* the capability byte for `OP_UTXOTOKENCATEGORY` and `OP_OUTPUTTOKENCATEGORY` — so the check must include `0x02` (minting) appended.[^chip-spec] Verifying only `OP_UTXOTOKENCATEGORY(0) == expectedCategory` (33 bytes for category+capability) catches this; verifying `expectedCategory` (32 bytes) without the capability suffix misses it.

CashScript's `tx.inputs[0].tokenCategory` returns the full category+capability concatenated, so high-level code tends to get this right. Hand-written Bitauth Auth Templates are where this slips.

## 7. Cauldron-specific risks

Cauldron's architecture is documented at length in `entity.cauldron-dex`. The security-relevant risks are:[^cauldron][^cashlab]

### 7.1 Pool operator key compromise

Each pool's bytecode is parameterized by a 20-byte `withdraw_pubkey_hash`. The operator (the address that created the pool) can sign a `withdraw_cauldron_poolv0` branch and pull out all the BCH and tokens in the pool. **A pool is only as safe as the operator's key custody.**[^cashlab]

The `cauldron_poolv0` template puts withdrawal on a Schnorr signature on the operator key. If that key leaks, the attacker withdraws everything; the constant-product check only runs on the swap branch (the depth-2 stack path), not on the withdrawal branch.

### 7.2 Pool bytecode mismatch

Because the canonical pool source is a Bitauth Auth Template (not a `.cash` file),[cashscript][^cashlab] the only way to verify a pool address is to:

1. Decode the address → extract the `withdraw_pubkey_hash` from the locking bytecode template's parameter slot.
2. Call `ExchangeLab.generatePoolV0LockingBytecode({withdraw_pubkey_hash})` from `@cashlab/cauldron`.
3. Hash the bytecode back to the same cashaddr.

A wallet that doesn't do this can't tell a Cauldron V0 pool from a phishing contract that *looks* like one. The SDK's `verifyPoolBytecode()` is the canonical check.

### 7.3 LP NFT under-collateralization (impermanent loss + rug)

Cauldron's LP NFT (the receipt the operator receives on add-liquidity) is just an NFT in the pool's category. Its market value is the LP's proportional claim. There's no on-chain enforcement that an LP's proportional claim matches what the pool contract thinks — the math is enforced at *deposit* time, but if the operator withdraws asymmetrically (withdrawing only one side), the remaining LPs' NFTs are diluted.

This is not a "bug" in Cauldron; it's the standard AMM-LP risk. But it's the dominant loss vector for Cauldron LPs, and it's worth flagging for the bot's treasury if we LP into PUSD/BCH.

### 7.4 Oracle manipulation (Cauldron uses no on-chain oracle, but adjacent protocols do)

Cauldron itself is a constant-product pool — no oracle. But PUSD's `PriceContract` uses an off-chain oracle signature (`oraclePubKey` in the constructor), and the oracle signature is a single point of failure. A compromised oracle pubkey can sign any price; the contract honors it (PUSD's bytecode-verification.md walks through the audit).

For Cauldron, the analogous risk is **price impact** on small pools: a large swap on a thin pool moves the price sharply, which a downstream contract reading the pool's effective price (via reserves) could be manipulated into acting on. This is "oracle manipulation via AMM reserves" — a known DeFi attack class, less acute on Cauldron because no major on-chain protocol reads reserves as a price feed.

## 8. PUSD / ROACH pattern analysis

### PUSD (ParyonUSD)

PUSD's `Borrowing` contract is the entry point: deposit BCH collateral, mint PUSD (the FT). The contract holds a **minting-NFT UTXO** of the PUSD category; every borrow spends it and recreates it, so the PUSD supply can only grow via the covenant. The pattern is:[^pusd]

- **Minting NFT** in covenant → spending the covenant mints new PUSD to the borrower's output.
- **Stability Pool** holds a separate mutable-NFT category (`POOL_CATEGORY_ID = 7708645a…`). When a user stakes PUSD, the contract reads the current epoch from `OP_UTXOTOKENCOMMITMENT(0)` of the pool input (`parseNextEpoch` in `bch-bot/lib/pusd.mjs:66`), then writes a receipt NFT with `epoch + amount` to the user's output.

**Risk surface:**

- The `Borrowing` contract holds the minting NFT; if the contract's logic is buggy (e.g. doesn't enforce min collateral ratio), the PUSD supply can be over-minted. The bytecode fingerprint (per `entity.cashscript`) is the canonical integrity check.
- The `PriceContract` has `migrateContract()` exposed. PUSD ships this with a `migrationKey`; the trust assumption is that the operator (Paryon team) doesn't migrate to a malicious bytecode. Users who don't trust the operator should not deposit.[^cashscript]
- The Stability Pool's `AddLiquidity.cash` requires a strict 5-6 output shape and a fixed sighash split (0x41 for P2PKH inputs, 0x61 for the covenant function input). If a wallet gets any output index wrong, the covenant fails. `bch-bot/lib/pusd.mjs:113` documents the exact shape and would catch index mistakes before broadcast.

### ROACH (Broach)

ROACH is a "pure-FT" category — `isFungible=true, hasNFT=false` per the cashtokenmarkets API.[^roach] There is no covenant or minting NFT; the FT was minted in a single genesis tx with the full 1B supply, and the category has no further issuance capability.

**Risk surface:**

- **None at the protocol level** — the category has no covenant, so there's no logic to bug out. The supply is fixed.
- **Holder trust** — the deployer wallet is the same wallet that will hold the Brainrot Battles treasury (per Luke's decision 2026-07-12). This creates a **shared-key risk**: if the ROACH deployer key leaks, the Brainrot minting NFT can also be stolen. Flagged for upgrade to 2-of-3 multisig post-launch.[^roach]
- **No minting NFT means no recovery from a category bug** — there's no admin key to fix anything. (This is actually a feature for trust minimization.)

## 9. Implementation pitfalls (bch-bot patterns)

### 9.1 Correct patterns

- **Token prefix from Electrum → libauth**: `lib/tokens.mjs:30` (`utxoToTokenPrefix`) maps `token_data.{category, amount, nft}` to `{category: Uint8Array, amount: bigint, nft?: {capability, commitment}}`. This is correct and matches Selene Wallet's `src/util/normalize.ts`.[^bot-tokens]
- **Dust bumping via libauth**: `getDustThreshold(output, DUST_RELAY_FEE)` is computed dynamically per output type. The default `satsAmount = 1000n` is safely above dust for any token-bearing P2PKH output including those carrying an NFT commitment.[^utxo-mempool]
- **Token-aware coin selection**: `selectInputsForTokenSend` (line 156) does FT-largest for the category, then BCH-largest from BCH-only UTXOs, never spending a token UTXO into fee.[^bot-tokens]
- **Capability-aware commitment building**: `lib/pusd.mjs:86` correctly distinguishes the padded 4-byte epoch portion from the minimal-BE amount portion.[^bot-pusd]
- **Minting-NFT recreation**: `lib/pusd.mjs:140` recreates the Stability Pool's minting NFT in output[0] with `capability: 'minting'`, preserving the input's commitment. Without this, the pool's minting capability would be destroyed and the pool becomes unstakeable.

### 9.2 Patterns to watch

- **`commitment || '00'.repeat(32)`** in `lib/tokens.mjs:93`: when the user doesn't supply a commitment, the bot falls back to 32 zero bytes. This is *fine* (a zero-length commitment is equivalent to `0x00…00` 32-byte commitment in token prefix encoding), but it's worth knowing — sending "no commitment" as 32 zeros versus a truly empty commitment produces two different on-chain commitments even though they parse identically at consensus. A covenant that checks `OP_OUTPUTTOKENCOMMITMENT(0) == OP_UTXOTOKENCOMMITMENT(0)` accepts both as equal because both opcodes push `0` for zero-length commitments.
- **No category validation on input**: `selectTokenUtxos` (line 129) filters by exact category match — good — but does not validate that the input UTXO has the expected capability. For an FT send this is fine; for an NFT send or a covenant input, the caller must verify the capability separately.
- **No minting-NFT burn protection in the bot's builders**: a tx the bot builds that intends to send a `minting` NFT to a regular P2PKH (i.e. forgetting to recreate it in a covenant output) would burn the category's minting capability forever. The bot currently has no `isMintingNftSpend()` guard. This is a **gap** — should be added if/when the bot ever handles minting NFTs directly.

### 9.3 Cashlab / libauth @cashlab/* patterns to verify

`@cashlab/cauldron`'s `generatePoolV0LockingBytecode({withdraw_pubkey_hash})` is the canonical bytecode generator.[^cashlab] The TypeScript signature requires `withdraw_pubkey_hash: Uint8Array(20)` (the RIPEMD-160 of the operator's compressed pubkey, not the pubkey itself). The bot's `lib/cauldron.mjs:70` does this correctly: it reads `rn.pkh` from Rostrum and passes it as `Uint8Array` — `hexToBin(rn.pkh)` produces a 20-byte buffer assuming Rostrum returned the hash, not the pubkey.

If a future version of Rostrum starts returning pubkeys directly, `generatePoolV0LockingBytecode` would compile a different locking bytecode, the address would change, and `verifyPoolBytecode()` would fail. This is a silent break that would manifest as "all my pools disappeared" after a Rostrum upgrade.

## 10. Gaps and known unknowns

- **No formal CashTokens CVE list exists** as of 2026-09-19 that I can cite. The closest is the BCHN release notes (v25.0.0, v27.0.0, v29.1.0) which mention CashTokens-specific consensus fixes, but these are node-side, not contract-side. A community-maintained list at `cashtokens.org/security` or similar does not exist.
- **The exact opcode-level behavior of `OP_UTXOTOKENCATEGORY` when input has FT but no NFT**: the spec says "push the UTXO's token category" — i.e. just 32 bytes, no capability byte. This is consistent across the test vectors but not extensively tested in third-party covenant libraries.
- **PUSD's `Borrowing.cash` and `AddLiquidity.cash` full source** is not in this repo; only the bot's intent-shaped builders. The fingerprint registry in `entity.cashscript` is the trust anchor until the audit reports are independently re-derived.
- **Cauldron V1 / pool upgrades**: as of 2026-09-19, only `cauldron_poolv0` exists. If Riften Labs ships a V1, the SDK's `generatePoolV1LockingBytecode` will produce incompatible bytecode, and pools minted against V0 won't be visible to V1-aware wallets.
- **The bot's `cauldron.mjs` does not call `verifyPoolBytecode()`** after generating pool bytecode from a Rostrum response — it trusts that `generatePoolV0LockingBytecode(pkh) == bytecode_at_address`. This is a gap.

## Sources

[^chip-spec]: CHIP-2021-05 CashTokens specification, v2.2.2 (2023-05-20). <https://cashtokens.org/docs/spec/chip/> — Token Categories, Token Types, Token Behavior, Token Validation Algorithm, Token Inspection Operations sections.
[^cashscript-covenants]: CashScript docs, "Writing Covenants & Introspection." <https://cashscript.org/docs/guides/covenants/> — `StreamingMecenas`, `LastWill`, `Mecenas` examples; introspection opcode reference.
[^cashscript]: `entity.cashscript` wiki page (cashc 0.13.0, fingerprint registry, migrateContract caveat). Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/entities/cashscript.md`.
[^p2sh32]: bitcoincashresearch.org, "P2SH32: a long-term solution for 80-bit P2SH collision attacks." <https://bitcoincashresearch.org/t/p2sh32-a-long-term-solution-for-80-bit-p2sh-collision-attacks/750>
[^electrs-cash]: Bitcoin Unlimited, "ElectrsCash." <https://github.com/BitcoinUnlimited/ElectrsCash> — Rust electrs fork; `blockchain.scripthash.listunspent` returns `token_data` for CashTokens-aware UTXOs.
[^electrum-trust]: Spark Research, "Electrum Server Implementations: Electrs, ElectrumX, and Fulcrum Compared." <https://www.spark.money/research/bitcoin-electrum-server-architecture>
[^utxo-mempool]: `reference.security.utxo-and-mempool` wiki page. Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/security/utxo-and-mempool.md`.
[^bot-tokens]: bch-bot `lib/tokens.mjs` (`utxoToTokenPrefix`, `createTokenOutput`, `createNftOutput`, `selectTokenUtxos`, `selectInputsForTokenSend`). Local: `/home/luke/bch-bot/lib/tokens.mjs`. Verified against Selene Wallet's `src/util/normalize.ts` and `src/kernel/bch/TransactionBuilderService.ts`.
[^bot-pusd]: bch-bot `lib/pusd.mjs` (`buildReceiptCommitment`, `parseNextEpoch`, `buildStakeTransaction`). Local: `/home/luke/bch-bot/lib/pusd.mjs`.
[^bot-sign]: bch-bot `lib/sign.mjs` (token prefix into signing serialization). Local: `/home/luke/bch-bot/lib/sign.mjs`.
[^cauldron]: `entity.cauldron-dex` wiki page. Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/entities/cauldron-dex.md`.
[^cashlab]: `sources/cashlab-cauldron.md` — cashlab cauldron_libauth_template.json, util.ts, exchange-lab.ts. Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/sources/cashlab-cauldron.md`.
[^pusd]: `entity.paryonusd` wiki page (token id, decimals, contracts repo). Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/entities/paryonusd.md`.
[^roach]: `entity.broach-token` wiki page (category id, isFungible/hasNFT, deployer wallet = treasury wallet). Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/entities/broach-token.md`.
[^brainrot]: `entity.brainrot-collection` wiki page (per-NFT commitment stores character index 0-19). Local: `/home/luke/.openclaw/workspace/memory-bch-wiki/entities/brainrot-collection.md`.

---

*Authored by security-research subagent, lens: CashTokens security. Cross-references `reference.security.utxo-and-mempool` (sender-side) and `entity.cashscript` (compiler/fingerprint). Word count: ~2,400.*


---

## Addendum: the real output format, and two node implementations (2026-10-02)

### The output layout, read off a node-accepted transaction

Decoded from a confirmed mainnet ROACH output (100 base units), not from the spec
and not from memory:

```
<35-byte token prefix> <25-byte P2PKH>
ef 53ff3501720c686780457d9affa6e60f552f5685bb6a768325f926a380ef2c8910 64  76a914…
```

- Marker is **`0xef`** (`PREFIX_TOKEN`) — the script's first byte, **not**
  `OP_RETURN` (0x6a). My first hypothesis was wrong about this.
- A 34-byte commitment, then the amount as a **CompactSize varint** (`64` = 100),
  then the locking bytecode. The prefix is **variable length**: the same shape
  with 206,636,482 units encodes it in 4 bytes (`fec205510c`).
- The prefix does **not** contain the category id in clear, and does not vary
  with the destination.

`token_bitfield` per CHIP-2022-02 `PREFIX_TOKEN`: high nibble is
`prefix_structure` (0x80 RESERVED, 0x40 HAS_COMMITMENT_LENGTH, 0x20 HAS_NFT,
0x10 HAS_AMOUNT), low nibble is the NFT capability (0 immutable, 1 mutable,
2 minting, >2 reserved). A real ROACH UTXO reports `16` = HAS_AMOUNT only.

**libauth encodes all of this correctly.** I asserted in this wiki that
`outputToLibauth` dropped the `token` field and produced a bare P2PKH; that was
wrong, and a test now pins the generated output **byte-for-byte** against the real
script above. The `bad-txns-vout-tokenprefix (code 16)` rejection was a *fee* bug
in disguise. A transaction rejected for one field is not evidence about a
different field.

### `listunspent` names token fields differently per server

| Server | Fields on `blockchain.scripthash.listunspent` |
|---|---|
| `cashnode.bch.ninja` (Fulcrum) | **none** |
| `rostrum.cauldron.quest` (Rostrum) | `has_token`, `token_id`, `token_amount`, `token_bitfield` |

Every consumer in the bch-bot codebase read `utxo.token_data.{amount,category}`.
Against a Rostrum response they all read `undefined`, so **the wallet reported
zero tokens while holding 2 confirmed ROACH.** Normalise at the network boundary
rather than teaching every call site about field-name variants.

### Amount bounds belong at construction, and fail closed

`createTokenOutput` / `createNftOutput` passed amounts and category ids straight
to libauth's encoder. `0`, negative, and 2^100 either travelled into the encoder
or threw a bare `BigInt` error naming neither the token nor the bound. Validate
where the intent is still known: FT amount `1..0xffffffffffffff7f`, category
exactly 64 hex chars, commitment hex and ≤ 40 bytes, capability one of
none/mutable/minting.
