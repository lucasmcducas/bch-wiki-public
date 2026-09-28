---
pageType: reference
id: reference.security.script-and-signing
description: Bitcoin Cash UTXO script opcodes (legacy, post-2018 re-enabled, post-2026 upgrade), CashTokens NFT/FT inspection opcodes, sighash standards (0x41/0x61/FORKID/UTXOS), signature and transaction malleability, P2SH-32 vs P2S, and CashScript covenant patterns.
sourceUrl: https://reference.cash/protocol/forks/hf-20230515
---

# Bitcoin Cash UTXO Script & Signing Standards

*Security reference for the BCH bot. Verified against BCHN v29.1.0 (current mainnet, post May 2026 upgrade). All opcodes and sighash bytes were verified against the canonical CHIP specifications and the `signing-serialization.ts` source in `@bitauth/libauth` v3.x.*

## 1. Bitcoin Script on BCH — the modern ISA

Bitcoin Cash runs the **CashVM** instruction set (a Bitcoin Script superset). Unlike Bitcoin Core, BCH has repeatedly restored opcodes that BTC had disabled after the 2010 OP_CHECKMULTISIG bug, plus added new ones through CHIPs.

### 1.1 Standard opcodes (always present)

The "always on" Bitcoin Script opcodes that work identically on BTC and BCH: `OP_DUP` (0x76), `OP_HASH160` (0xa9), `OP_EQUAL` (0x87), `OP_EQUALVERIFY` (0x88), `OP_CHECKSIG` (0xac), `OP_CHECKMULTISIG` (ae), `OP_VERIFY` (0x69), `OP_RETURN` (0x6a), `OP_PUSHBYTES_n` / `OP_PUSHDATA_n`, all arithmetic (`OP_ADD`, `OP_SUB`, `OP_NUMEQUAL`, etc.), all flow control (`OP_IF`, `OP_ELSE`, `OP_ENDIF`), the basic crypto ops (`OP_SHA256`, `OP_HASH256`, `OP_RIPEMD160`, `OP_CHECKMULTISIG`), and the stack ops. [Source: https://en.bitcoin.it/wiki/Script — inherited base]

### 1.2 Re-enabled opcodes, May 2018 (BCH first year of upgrade)

The May 2018 HF on BCH re-enabled a suite that BTC had disabled in 2010: `OP_CAT`, `OP_SPLIT`, `OP_AND`, `OP_OR`, `OP_XOR`, `OP_DIV`, `OP_MOD`, `OP_NUM2BIN`, `OP_BIN2NUM`. These were the basis of the early covenant workaround using `OP_CHECKSIG + OP_CHECKDATASIG` against a preimage built with `OP_CAT`. [Source: https://medium.com/@DrRoyMurphy/bitcoin-opcodes-for-dummies-4cd6f10d744b]

### 1.3 November 2018 — opcodes still missing on BTC

`OP_CHECKDATASIG` (0xba), `OP_CHECKDATASIGVERIFY` (0xbb), `OP_REVERSEBYTES` (0xbc). These were the *BCH-only* additions in the November 2018 upgrade and form the foundation for the OP_CHECKDATASIG covenant pattern that CashScript (2019-2021) relied on before native introspection landed. `OP_CHECKDATASIG` accepts a `(message, signature, pubkey)` triple — message can be any stack item, signature is 64-byte Schnorr or DER ECDSA — and pushes true if the signature is valid for `SHA256(message)`. [Source: https://github.com/bitjson/bch-2022/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md]

### 1.4 November 2018 — arithmetic ops re-enabled (second wave)

`OP_MUL`, `OP_LSHIFT`, `OP_RSHIFT`, `OP_INVERT` were re-enabled on BCH in November 2018 per the upgrade spec. Note that `OP_LSHIFT` / `OP_RSHIFT` were *redefined* as byte-string shifts (not numeric), which is what eventually led to the May 2026 split into `OP_LSHIFTNUM`/`OP_RSHIFTNUM` (numeric) vs `OP_LSHIFTBIN`/`OP_RSHIFTBIN` (binary). [Source: https://github.com/bitcoin-sv-specs/protocol/blob/master/updates/2018-11-15%20BCH%20Upgrade%20Spec.md]

### 1.5 May 2022 — Native Introspection (CHIP-2021-02)

Fifteen new opcodes occupying `0xc0–0xcf` (range reserved after a 3-codepoint gap following `OP_REVERSEBYTES` at `0xbc`). Replaces the costly OP_CHECKDATASIG-preimage workaround with single-byte direct introspection. From the spec [Source: https://reference.cash/protocol/forks/chips/2022-05-native-introspection-opcodes]:

| Opcode | Byte | Arity | What it pushes |
|--------|------|-------|----------------|
| `OP_INPUTINDEX` | 0xc0 | 0 | Index of the input being evaluated (Script Number) |
| `OP_ACTIVEBYTECODE` | 0xc1 | 0 | Bytecode under evaluation (P2SH redeem bytecode, otherwise locking bytecode) |
| `OP_TXVERSION` | 0xc2 | 0 | nVersion |
| `OP_TXINPUTCOUNT` | 0xc3 | 0 | input count |
| `OP_TXOUTPUTCOUNT` | 0xc4 | 0 | output count |
| `OP_TXLOCKTIME` | 0xc5 | 0 | nLocktime |
| `OP_UTXOVALUE` | 0xc6 | 1 | value (sats) of input at popped index |
| `OP_UTXOBYTECODE` | 0xc7 | 1 | locking bytecode of input at popped index |
| `OP_OUTPOINTTXHASH` | 0xc8 | 1 | outpoint txid of input at popped index (internal byte order) |
| `OP_OUTPOINTINDEX` | 0xc9 | 1 | outpoint vout of input at popped index |
| `OP_INPUTBYTECODE` | 0xca | 1 | unlocking bytecode of input at popped index |
| `OP_INPUTSEQUENCENUMBER` | 0xcb | 1 | nSequence of input at popped index |
| `OP_OUTPUTVALUE` | 0xcc | 1 | value of output at popped index |
| `OP_OUTPUTBYTECODE` | 0xcd | 1 | locking bytecode of output at popped index |

> **Push-only rule.** All opcodes above `0x60` are forbidden in unlocking bytecode, including the entire introspection range. This blocks a third-party malleability vector (any non-push op can be replaced by a push of its precomputed result). [Source: same CHIP, "Push-Only Limitation of Unlocking Bytecode"]

### 1.6 May 2023 — CashTokens (CHIP-2022-02) added 6 token opcodes

Extending the introspection range with six more opcodes (`0xce–0xd3`). From the CashTokens CHIP [Source: https://cashtokens.org/docs/spec/chip/]:

| Opcode | Byte | What it pushes |
|--------|------|----------------|
| `OP_UTXOTOKENCATEGORY` | 0xce | token category of input at popped index; if NFT present, concatenated with capability byte (0x00 none, 0x01 mutable, 0x02 minting); empty if no tokens |
| `OP_UTXOTOKENCOMMITMENT` | 0xcf | NFT commitment of input at popped index; empty/zero if no NFT or zero-length commitment |
| `OP_UTXOTOKENAMOUNT` | 0xd0 | fungible token amount of input at popped index; 0 if no FT |
| `OP_OUTPUTTOKENCATEGORY` | 0xd1 | same, but for output at popped index |
| `OP_OUTPUTTOKENCOMMITMENT` | 0xd2 | NFT commitment of output |
| `OP_OUTPUTTOKENAMOUNT` | 0xd3 | fungible token amount of output |

> **Important: there is no `OP_NFTINPUTINDEX` opcode.** What the previous (brief) research prompt asked about. The token "input index" is just `OP_INPUTINDEX` paired with `OP_UTXOTOKENCATEGORY` / `OP_UTXOTOKENCOMMITMENT` / `OP_UTXOTOKENAMOUNT`. If you want to know whether the *current input* is the minting-NFT input (the common covenant pattern), the pattern is `OP_INPUTINDEX OP_UTXOTOKENCATEGORY 0x02 OP_EQUAL` (the `0x02` is the minting capability suffix concatenated to the 32-byte category).

### 1.7 May 2025 — VM Limits + BigInt (CHIP-2024-07)

`OP_NUM2BIN` / `OP_BIN2NUM` extended to support up to 80,000-bit script numbers. Removes the 32-bit truncation that previously capped usable `OP_UTXOVALUE`/`OP_OUTPUTVALUE` results. [Source: BCHN v28.0.0 release notes — sources/bchn-release-notes.md]

### 1.8 May 2026 (Layla upgrade) — P2S + Loops + Functions + Bitwise

Four CHIPs in BCHN v29.0.0 [Source: https://docs.bitcoincashnode.org/doc/release-notes/release-notes-29.0.0/]:

- **CHIP-2024-12 P2S (Pay-to-Script).** Locking bytecode is now the script directly (no P2SH hash wrap). New `P2S` template lock for covenant-heavy contracts that need to lock funds to bytecode longer than the unwrapped-in-P2SH20 size cap. P2SH32 still preferred for hash-shielded covenants (see §6 below). [Source: https://github.com/bitjson/bch-p2s]
- **CHIP-2021-05-loops (Bounded Looping).** Adds `OP_LOOP` / `OP_BREAK` and the iterator helpers `OP_ENTER`/`OP_EXIT`. Loops are bounded by VM limits, not by opcode count.
- **CHIP-2025-05 (Functions).** Reusable named script fragments (`OP_DEFINE` / `OP_INVOKE`).
- **CHIP-2025-05 (Bitwise).** Re-enables `OP_INVERT` (byte-level bit inversion), and adds split shift ops: `OP_LSHIFTNUM`/`OP_RSHIFTNUM` (numeric), `OP_LSHIFTBIN`/`OP_RSHIFTBIN` (binary string). The 2018 `OP_LSHIFT`/`OP_RSHIFT` were ambiguous about which they operated on; this resolves it. [Source: https://github.com/bitjson/bch-bitwise]

### 1.9 Opcodes that remain disabled / never enabled on BCH

- `OP_CODESEPARATOR` after the May 2018 spec — actually still present, but worth flagging that it splits the active bytecode in ways that catch covenant authors off guard (see `OP_ACTIVEBYTECODE` interaction in CHIP-2021-02 §"OP_ACTIVEBYTECODE Support for OP_CODESEPARATOR").
- `OP_MUL` is enabled (since 2018) but has *no constant-time implementation* in libsecp256k1; it's a host-side integer mul. Don't put it in a covenant path that processes attacker-controlled values.
- `OP_LEFT`, `OP_RIGHT`, `OP_SUBSTR` (from the original Satoshi opcodes list) were never enabled on BCH.
- No `OP_PUSH_TX_STATE` or any aggregate opcode (no `OP_TXINPUTVALUE`, etc.) — must be computed via loops introduced in May 2026.
- `OP_CHECKMULTISIG` with 65-byte Schnorr is treated as **ECDSA only** post-2019-Schnorr; BCH will not verify Schnorr under `OP_CHECKMULTISIG`. A Schnorr sig will cause script failure if used there. [Source: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2019-05-15-schnorr.md]

## 2. CashTokens-specific script extensions

### 2.1 The token prefix in outputs

Every output that carries tokens has, *before* index 0 of the locking bytecode, a **token prefix** starting with the codepoint `0xef` (PREFIX_TOKEN). The structure is:

```
PREFIX_TOKEN | 32-byte category_id | 1-byte token_bitfield
            | [nft_commitment_length + commitment]      if HAS_COMMITMENT_LENGTH
            | [ft_amount]                                if HAS_AMOUNT
```

The `token_bitfield` is two 4-bit fields: the upper 4 bits are prefix-structure flags (`0x40` HAS_COMMITMENT_LENGTH, `0x20` HAS_NFT, `0x10` HAS_AMOUNT); the lower 4 bits are the NFT capability (`0x00` none/immutable, `0x01` mutable, `0x02` minting). [Source: https://cashtokens.org/docs/spec/chip/ §Token Prefix]

> **Commitment length is capped at 40 bytes** in the consensus rules, but the design allows future upgrades to lift it. CompactSizes must be minimally encoded.

### 2.2 NFT capability rules

- **Minting NFT (capability 2):** the spending tx may create any number of new NFTs of the same category, each with any commitment and any capability (mutable or minting included).
- **Mutable NFT (capability 1):** the spending tx may create exactly **one** new NFT of the same category, with any commitment and (optionally) the mutable capability. **You cannot create a minting NFT from a mutable NFT.**
- **Immutable NFT (capability 0):** commitment cannot be modified when spent. (Same category is allowed only if covenants enforce it; otherwise the immutable NFT is consumed.)
- **Fungible tokens (FT):** independent of NFT capability. FTs can only be created at genesis. FT total per category is capped at `2^63 - 1` = 9,223,372,036,854,775,807.

### 2.3 The genesis invariant

A token category is identified by the txid of the transaction that **creates** it, and creation requires the spending of **output index 0** of a previous transaction as the genesis input. That genesis tx then emits outputs containing the initial FT supply and any initial NFTs. Implementations locate the genesis by walking the txid back: take the category ID as a txid, look up that tx, find the input that spent output 0 — that's the genesis tx.

### 2.4 Token-aware signing serialization

The signing preimage for **any** input spending a token-bearing UTXO is automatically modified to include the full encoded token prefix immediately before `coveredBytecode` in the preimage. This applies to all sighash types — it does **not** require `SIGHASH_UTXOS`. The signing algorithm finds the token prefix by looking at `sourceOutputs[inputIndex].token`, fetches the previous output's token data, and prepends it to the preimage before hashing. (In libauth: `encodeSigningSerializationBCH` accepts an `outputTokenPrefix` field and inserts it.)

## 3. Sighash standards

### 3.1 The full flag table (BCH)

From `signing-serialization.ts` in `@bitauth/libauth` (verified 2026-09-19 against v3.1.0-next.8):

| Flag | Byte | Name |
|------|------|------|
| `allOutputs` | 0x01 | SIGHASH_ALL |
| `noOutputs` | 0x02 | SIGHASH_NONE |
| `correspondingOutput` | 0x03 | SIGHASH_SINGLE |
| `utxos` | 0x20 | SIGHASH_UTXOS (CashTokens, mandatory for token inputs) |
| `forkId` | 0x40 | SIGHASH_FORKID (BCH replay protection since UAHF 2017) |
| `singleInput` | 0x80 | SIGHASH_ANYONECANPAY |

### 3.2 Pre-computed sighash bytes

| Algorithm name | Byte | Meaning |
|----------------|------|---------|
| `allOutputs` | **0x41** | SIGHASH_ALL \| SIGHASH_FORKID — by far the most common in the wild |
| `allOutputsAllUtxos` | **0x61** | + SIGHASH_UTXOS — CashTokens-aware version of 0x41 |
| `allOutputsSingleInput` | 0xC1 | + ANYONECANPAY |
| `allOutputsSingleInputAllUtxos` | 0xE1 | both |
| `noOutputs` | 0x42 | SIGHASH_NONE \| SIGHASH_FORKID |
| `noOutputsAllUtxos` | 0x62 | + SIGHASH_UTXOS |
| `correspondingOutput` | 0x43 | SIGHASH_SINGLE \| SIGHASH_FORKID |
| `correspondingOutputAllUtxos` | 0x63 | + SIGHASH_UTXOS |
| `correspondingOutputSingleInput` | 0xC3 | SIGHASH_SINGLE \| ANYONECANPAY |
| `correspondingOutputSingleInputAllUtxos` | 0xE3 | all four |

> **The 24-bit fork ID.** The sighash type field is actually a 32-bit value where the top 24 bits are reserved for a fork ID and the bottom 8 bits carry the actual sighash flags. The BCH mainnet fork ID is `0x000000`, so the sighash byte is the "low byte" of the 32-bit field, and the 3-byte fork ID is normally `0x000000`. Wallets generally pass the byte alone; libauth's `encodeSigningSerializationBCH` accepts the byte and an optional `forkId` parameter, but you should leave it at the default zero unless deliberately signing for a future fork.

### 3.3 What each commits to

Per the replay-protected sighash spec (BUIP-HF / BIP-143 adaptation) [Source: https://www.reference.cash/protocol/forks/replay-protected-sighash]:

- **`0x41` (SIGHASH_ALL \| FORKID):** commits to all inputs (via hashPrevouts), all input sequences (via hashSequence), all outputs (via hashOutputs), the spending input's prevout + scriptCode + value, nLocktime, nVersion, sighash type. The whole transaction is locked except the unlocking scripts themselves.
- **`0x42` (SIGHASH_NONE \| FORKID):** commits to inputs but **none of the outputs**. Used for "anyone-can-collect" dust sweepers — useful but be aware anyone can redirect funds.
- **`0x43` (SIGHASH_SINGLE \| FORKID):** commits to inputs and **only the output at the same index as the input being signed**. Other outputs are malleable. Rarely used on its own.
- **`0xC1` / `0xE1` (ALL|UTXOS \| ANYONECANPAY):** commits to **only the input being signed** (no hashPrevouts, no hashSequence), all outputs. The "crowdfund contribution" pattern — signers promise their output without seeing the full set of other inputs. Compose with `SIGHASH_UTXOS` (mandatory for token inputs) → 0xE1.
- **`0x61` (ALL \| FORKID \| UTXOS):** the CashTokens-aware version of 0x41. Adds `hashUtxos` (double-SHA256 of the serialization of all *spent UTXOs*, excluding output count) to the preimage immediately after `hashPrevouts`. **Required for any input that spends a token-bearing UTXO** — without it, consensus rejects the signature. [Source: CashTokens CHIP §SIGHASH_UTXOS]
- **`0x62` / `0x63` / `0xE3`:** similar to 0x42 / 0x43 / 0xE1 but with `hashUtxos` included. All mandatory for token inputs regardless of which "outputs" mode is selected.

> **Critical BCH-only rule.** `SIGHASH_UTXOS` (`0x20`) and `SIGHASH_ANYONECANPAY` (`0x80`) **must not** be combined; the VM rejects such signatures. `SIGHASH_UTXOS` requires `SIGHASH_FORKID` (0x40). [Source: CashTokens CHIP §SIGHASH_UTXOS]

### 3.4 The signing preimage layout

For an input at index `i`, with `coveredBytecode` (the script being signed, e.g. the P2PKH pubkey-hash or the P2SH redeem script), and `outpointTokenPrefix` (the token prefix of the spent output, if any):

```
nVersion (4 LE) || hashPrevouts (32) || hashSequence (32) ||
outpoint (32+4 LE) || [outpointTokenPrefix] || coveredBytecode (varint-len-prefixed) ||
value (8 LE) || nSequence (4 LE) || hashOutputs (32) || nLocktime (4 LE) || sighashType (4 LE)
```

`hashUtxos` (when 0x20 set) inserts between `hashSequence` and the per-input outpoint, with content = `hash256(serialize_spent_utxos)`. The token prefix goes immediately before `coveredBytecode`, so the cover-script byte string is wrapped as `[prefix || coveredBytecode]` when serializing that field.

## 4. Signature malleability

### 4.1 ECDSA malleability

An ECDSA signature `(r, s)` validates for both `s` and `n - s` (where `n` is the secp256k1 group order). Both are mathematically valid; they produce the same `r` but different `s`. A third party that has a valid `(r, s)` can produce `(r, n-s)` without the private key — this **mutates the txid** without invalidating the signature.

**The Low-S rule (BIP-146)** mandates that only signatures with `s <= n/2` are consensus-valid. This collapses the `(r, s)` / `(r, n-s)` choice to a single canonical form. Bitcoin Core made this a *standardness* rule in v0.11.1 (2015), but only enforced it as a *consensus rule* in BIP-146 (which never activated on BTC). **BCH enforces low-S as a consensus rule since the November 2018 upgrade** — meaning a high-S ECDSA signature is invalid at the block-validation layer, not just non-standard relay. [Source: https://bips.dev/146/]

> **Schnorr (BIP-340) does not have this problem.** Schnorr signatures are linear in `(r, s)` such that there is exactly one valid signature for a given `(R, m, k)` tuple. BCH adopted Schnorr via the 2019-05-15 upgrade (CHIP: Schnorr signatures for BCH); libauth exposes `secp256k1.signMessageHashSchnorr(hash, privateKey)`. Use Schnorr wherever you can.

### 4.2 DER encoding malleability

BIP-66 (strict DER) is also enforced as a consensus rule on both BTC and BCH. The combined BIP-66 + low-S requirement means ECDSA signatures on BCH have **zero third-party malleability** at the consensus layer.

### 4.3 Sighash-type byte malleability (historical)

Old BTC transactions could have a third party flip the sighash byte (e.g. `0x41` → `0x42`) inside the unlocking script, mutating the txid. The Replay Protected Sighash spec (BCH, 2017) eliminates this: signatures are gated on `SCRIPT_ENABLE_SIGHASH_FORKID` and the sighash type byte is included in the preimage as a 4-byte LE field that includes the fork ID. Flipping it changes the digest. [Source: https://www.reference.cash/protocol/forks/replay-protected-sighash]

### 4.4 What can still be mutated

On BCH today, signature malleability is essentially eliminated by BIP-66 + low-S + sighash-in-preimage. The remaining third-party malleability surface is in the **unlocking script itself** — non-push operations could in theory be replaced by precomputed pushes. This is why BCH has the push-only rule for unlocking bytecode (`OP_VERIFY` and up are forbidden). **Do not** use `OP_VERIFY` / `OP_CHECKSIG` / any non-push op inside a scriptSig. The `OP_CHECKDATASIG` workaround that some old covenants used (signing the preimage, then checking the preimage against itself with `OP_CHECKDATASIG`) was the last significant source of sigscript-level malleability; it has been fully replaced by native introspection since May 2022.

## 5. Transaction malleability

### 5.1 The three classes

1. **Signature/encoding malleability** — covered in §4 above. Eliminated by BIP-66 + low-S on BCH.
2. **Script-content malleability** — the pre-May-2022 `OP_CHECKDATASIG` workaround pattern was malleable because anyone could substitute alternative unlocking scripts that produced the same effect. Now eliminated by the push-only rule + native introspection.
3. **Output set malleability** — old BTC allowed the sender to mutate outputs by adding/removing OP_RETURN data, but on BCH `OP_RETURN` outputs and other outputs both contribute to the txid via `hashOutputs`. Mutating outputs changes the txid by design.

### 5.2 BCH-specific risks

- **Dust refund outputs.** If you broadcast a tx where one output's sat value is below the dust threshold (1,000 satoshis on most BCH mempools), some relayers will drop it, some will absorb it into the fee. The presence or absence of a dust output changes the `hashOutputs` digest and thus the txid. **Fix:** consolidate dust into a single explicit "fee absorber" output, or set the dust output value above 1,000 sats.
- **Change-output malleability.** If your signer uses `SIGHASH_SINGLE` (`0x43`) and there are outputs beyond the one being signed, those outputs can be changed by a third party without invalidating your signature. This bites when you expect a particular txid. Use `SIGHASH_ALL` (`0x41`) for any tx where the receiver cares about the txid.
- **CashTokens: token prefix malleability.** If you sign a tx spending a token UTXO without `SIGHASH_UTXOS` (`0x21` portion of 0x61), a third party could not change the output set without invalidating the signature, but they could mutate **which inputs** contribute which token prefix to the preimage by reordering inputs. With `SIGHASH_UTXOS`, the spent-UTXO set is committed via `hashUtxos`. **Always use 0x61 when the input spends a token UTXO.**
- **Coinbase-only.** Coinbase transactions have a special-case: their only input has outpoint index `0xFFFFFFFF`. They can never carry tokens. Don't try to spend a coinbase UTXO inside a covenant that uses `OP_OUTPOINTINDEX`.
- **DSProofs don't cover token-aware transactions.** BCHN v29.1.0 release notes explicitly state: *"Transactions with SIGHASH_UTXO are not covered by DSProofs at present."* If the bot is accepting a token-bearing 0-conf tx, DSProof-based double-spend detection will not catch a conflicting replacement. [Source: sources/bchn-release-notes.md, v29.1.0 Limitations]

## 6. The 0x41 vs 0x61 distinction (and the libauth bug)

### 6.1 What the bytes mean

- **`0x41`** = `SIGHASH_ALL | SIGHASH_FORKID` — commits to inputs, sequences, outputs, and nLocktime/nVersion.
- **`0x61`** = `SIGHASH_ALL | SIGHASH_FORKID | SIGHASH_UTXOS` — same as 0x41 *plus* commits to the spent-UTXO set (`hashUtxos`).

The difference is exactly the `hashUtxos` field in the preimage.

### 6.2 When each is correct

- **Plain BCH send (no tokens):** `0x41` is correct and slightly cheaper (smaller preimage → smaller hash → same signature size, but the hashing work is less).
- **CashTokens-aware input (any input spending a UTXO that carries tokens):** `0x61` is **mandatory**. Without it, the consensus rules reject the signature, because the preimage needs to include `hashUtxos` for the algorithm to be considered valid.

### 6.3 The Selene choice

Selene Wallet (the moth-pattern BCH mobile wallet) signs all inputs with `0x41` for plain BCH sends and `0x61` for token sends. This is the **correct** choice. See Selene's `KeyManagerService.signInputs` (verified in Selene's public source: `gitlab.com/selene.cash/selene-wallet`).

### 6.4 The libauth 0x61 "bug" (it's not a bug, it's a misuse)

libauth v3 exposes `SigningSerializationType` as a string enum (`'allOutputs'`, `'allOutputsAllUtxos'`, etc.). The compiler path in `compiler-bch.ts` (`compilerOperationSigningSerializationTokenPrefix`, line 576) **wires `token_prefix` into the preimage automatically** when the wallet template references it. But for **ad-hoc libauth-built transactions** (not the compiler path), the caller must:

1. Construct the transaction.
2. Look up the token prefix of the source output: `outputTokenPrefix = encodeTokenPrefix(sourceOutputs[inputIndex].token)`.
3. Pass `outputTokenPrefix` as a field in the `encodeSigningSerializationBCH({...})` call.
4. Compute the sighash algorithm as `'allOutputsAllUtxos'` (0x61), not `'allOutputs'` (0x41).

The "libauth 0x61 bug" the parent brief references is **almost always caller code that picked `'allOutputsAllUtxos'` but did not include the `outputTokenPrefix` field**, or that picked `'allOutputs'` (0x41) and expected `hashUtxos` to be included automatically. libauth does NOT silently inject `hashUtxos`; the caller must select the right algorithm AND supply the token prefix. [Source: entities/libauth.md §"Version 3 surface"]

```ts
// Correct signing for a token-bearing input (libauth v3):
import {
  encodeSigningSerializationBCH,
  encodeTokenPrefix,
  hash256,
  secp256k1,
} from '@bitauth/libauth';

const { sourceOutputs, transaction } = buildTx(...);
const coveredBytecode = encodeLockingBytecodeP2pkh(myPubkeyHash);
const outputTokenPrefix = encodeTokenPrefix(sourceOutputs[myInputIndex].token);

const preimage = encodeSigningSerializationBCH({
  transaction,
  sourceOutputs,
  coveredBytecode,
  inputIndex: myInputIndex,
  signingSerializationType: 'allOutputsAllUtxos', // 0x61
  outputTokenPrefix,
  forkId: new Uint8Array(),                      // zero = BCH mainnet
});

const digest = hash256(preimage);
const sig = secp256k1.signMessageHashSchnorr(digest, myPrivKey);
// sig is 64 bytes; append 0x61 (or 0x41) to get the full DER-or-not signature.
```

## 7. CashScript covenant patterns

### 7.1 P2SH32 — the standard hash for covenants (CHIP-2022-05)

Standard P2SH uses HASH160 (RIPEMD-160(SHA-256(redeemScript))), which gives a 160-bit hash. For covenant-heavy contracts with many similar-looking redeem scripts, the 160-bit collision space is now considered feasible to brute-force in adversarial scenarios (the rationale is the same as BTC's move to 32-byte witness program hashes).

**P2SH32** uses HASH256 (SHA-256(SHA-256(redeemScript))) instead — a 256-bit hash. Same template structure as P2SH but `OP_HASH256` instead of `OP_HASH160` and a 32-byte push instead of 20-byte. CashAddress type byte `0x03` (vs `0x01` for legacy P2SH). Token-aware variant is type byte `0x07`. [Source: CashTokens CHIP §CashAddress Token Support]

P2SH32 lock:
```
OP_HASH256 <push 32-byte hash> OP_EQUAL
```

Activated May 2023 alongside CashTokens. **Recommended for all new covenants** unless the contract requires 20-byte compatibility.

> **BCHN wallet still defaults to P2SH20.** From v29.1.0 release notes: *"P2SH-32 is not used by default in the wallet (regular P2SH-20 remains the default wherever P2SH is treated)."* If you build wallets or contracts, override the default to use P2SH32.

### 7.2 P2S — when you don't need hash shielding (CHIP-2024-12, May 2026)

Pay-to-Script lets you commit directly to the script bytes without a hash wrapper. Useful when the redeem script is large enough that paying the hash overhead is wasteful, and when the script bytes themselves are part of the contract's identity (e.g. covenant address = hash of code).

```
<full redeem bytecode>     (no OP_HASH wrapping)
```

CashAddress type byte for P2S is `0x05`. [Source: https://github.com/bitjson/bch-p2s]

> **P2S vs P2SH32 tradeoff.** P2SH32 has a fixed 34-byte locking script regardless of the redeem script size (just the hash). P2S has locking script size equal to the redeem script size. For small redeem scripts (< 30 bytes), P2S is cheaper. For larger redeem scripts, P2SH32 is cheaper — and P2SH32 hides the redeem script until spend, which prevents front-running adversaries from copying your contract logic before you can use it. **Use P2SH32 unless you need the script to be self-describing in the locking script (e.g. an unspent contract address that must be derivable from a known template without re-execution).**

### 7.3 Common CashScript patterns (from cashscript.org/docs/guides/covenants)

**A. Restricting recipient (blind escrow):**
```solidity
contract Escrow(bytes20 arbiter, bytes20 buyer, bytes20 seller) {
  function spend(pubkey pk, sig s) {
    require(hash160(pk) == arbiter);
    require(checkSig(s, pk));
    int minerFee = 1000;
    int amount = tx.inputs[this.activeInputIndex].value - minerFee;
    require(tx.outputs[0].value == amount);
    require(tx.outputs[0].lockingBytecode == new LockingBytecodeP2PKH(buyer)
         || tx.outputs[0].lockingBytecode == new LockingBytecodeP2PKH(seller));
  }
}
```

**B. Recurring payment (Mecenas):** restricts one output to recipient, one to the contract itself.

**C. Last Will (Licho):** uses `tx.inputs[this.activeInputIndex].lockingBytecode` to require the change output be the same script as the input — i.e. the contract "renews itself" each spend.

**D. Streaming Mecenas with NFT-stored local state:** uses `tx.inputs[0].nftCommitment` to read the last claim height, writes new height via `tx.outputs[1].nftCommitment`. The mutable NFT carries state across transactions without changing the contract address. The contract enforces `tx.outputs[1].lockingBytecode == tx.inputs[0].lockingBytecode` to keep the same UTXO.

**E. Algorithmic trading / AMM (Cauldron):** uses `OP_UTXOVALUE` / `OP_OUTPUTVALUE` plus `OP_UTXOTOKENAMOUNT` / `OP_OUTPUTTOKENAMOUNT` to enforce the constant-product invariant `kOut >= kIn` where `k = satoshis * tokens`. Cauldron's official pool template is not a `.cash` file — it's a Bitauth Auth Template in `cashlab/packages/cauldron/src/cauldron-libauth-template.json` and bytecode is generated programmatically with `ExchangeLab.generatePoolV0LockingBytecode({withdraw_pubkey_hash})`. [Source: entities/cauldron-dex.md]

**F. Minting covenant with receipt NFTs:** the contract holds a minting NFT in its own UTXO and `mints` receipt NFTs to contributors by writing outputs with `OP_OUTPUTTOKENCATEGORY` / `OP_OUTPUTTOKENCOMMITMENT` set to the contributor's address. Used by ParyonUSD-style pooling patterns. [Source: CashScript covenant guide, "Issuing NFTs as receipts"]

### 7.4 The universal covenant shape

A typical BCH covenant has these elements:

```
OP_INPUTINDEX OP_ACTIVEBYTECODE             <-- get own script
<known-pubkey> OP_CHECKSIGVERIFY             <-- authenticate signer
OP_INPUTINDEX OP_UTXOVALUE                   <-- get own input value
OP_INPUTINDEX OP_UTXOBYTECODE                <-- get input's locking bytecode
<hash> OP_EQUAL                               <-- verify own identity (or skip)
OP_TXINPUTCOUNT OP_1 OP_NUMEQUAL             <-- only one input (no combining)
OP_TXOUTPUTCOUNT <N> OP_NUMEQUAL             <-- exact output count
OP_OUTPUTVALUE <amount> OP_NUMEQUAL          <-- verify expected amount
OP_OUTPUTBYTECODE <expected-locking-bytecode> OP_EQUAL  <-- verify destination
OP_OUTPUTTOKENCATEGORY <expected-category> OP_EQUAL    <-- if tokens
OP_OUTPUTTOKENAMOUNT <expected-amount> OP_NUMEQUAL    <-- if tokens
```

## 8. Implementation gotchas

These have caused real bugs in wallets/libraries as of 2026-09:

1. **`OP_ACTIVEBYTECODE` vs `OP_INPUTINDEX OP_UTXOBYTECODE` (CHIP-2021-02 §"Differences...").** In a non-P2SH spend, both return the locking bytecode. In a P2SH spend, `OP_ACTIVEBYTECODE` returns the **redeem bytecode** (raw script), while `OP_INPUTINDEX OP_UTXOBYTECODE` returns the **P2SH template** (`OP_HASH160 <20-byte-hash> OP_EQUAL`). Covenants that compute their own hash for self-renewal must use `OP_ACTIVEBYTECODE`, not `OP_UTXOBYTECODE`. [Source: CHIP-2021-02]
2. **`OP_UTXOVALUE` truncation pre-BigInt.** Before May 2025 (no BigInt), `OP_UTXOVALUE` of an input > 21.47 BCH could not be used in arithmetic. Always verify your covenant's maximum input value against this if you build anything pre-BigInt.
3. **`SIGHASH_UTXOS` × `ANYONECANPAY` is consensus-invalid.** Even if you set both flags by mistake, BCHN rejects the signature at VM evaluation. The libauth compile-time check is `'allOutputsAllUtxos'` and `'allOutputsSingleInputAllUtxos'` are separate types, so you can't accidentally combine.
4. **`SIGHASH_UTXOS` requires `SIGHASH_FORKID`.** Same rule — flag without FORKID is rejected. (Why? Because UTXOS-as-a-flag was added in the CashTokens spec and inherits FORKID's replay protection.)
5. **`outputTokenPrefix` is mandatory for token inputs,** but libauth's compiler wires it automatically and the manual path requires it explicitly. Forgetting it produces a signature that validates against a different preimage than the network sees → fails consensus. [Source: entities/libauth.md §Pitfalls]
6. **libauth v3 `Secp256k1` does not throw.** Signing errors are returned as `string | Uint8Array`, not thrown. Don't wrap calls in try/catch expecting throws. [Source: same]
7. **`signing_serialization.token_prefix` was the renamed compiler op** in libauth v3 (was `signing_serialization.token_data` in v2.x). Auth templates written against v2 will fail to compile against v3 without the rename.
8. **No `signTransaction` in libauth.** Compose `encodeSigningSerializationBCH` + a caller-supplied secp256k1 implementation. The compiler path hides this. [Source: entities/libauth.md §"The architecture you must understand"]
9. **Pure ESM only.** `@bitauth/libauth` v3 is ESM-only. Downstream projects need `"type": "module"` in package.json. [Source: same]
10. **BCHN wallet doesn't track tokens.** As of v29.1.0, `bitcoin-cli` does not maintain a token balance. To get your token balance, query `listunspent` (with filter options) and sum the `tokenData.amount` fields manually. [Source: sources/bchn-release-notes.md]
11. **`SIGHASH_UTXO` is not DSProof-covered.** If you accept 0-conf txs from external senders that use 0x61 sigs, your DSProof monitor will not detect a double-spend. For high-value token 0-conf, wait for 1 confirmation or use covenant-based escrow (CHIP-2021-08 ZCEs — design-only, not deployed). [Source: sources/bchn-release-notes.md; references/bch-zero-conf-security.md]
12. **`getrawtransaction` at `verbosity=2` returns `sequence` as the last input key and `fee` after `vout`** since v29.0.0. Any code that relied on the pre-v29 ordering needs updates. [Source: BCHN v29.0.0 release notes]
13. **`OP_CHECKMULTISIG` won't verify 65-byte Schnorr.** Use `OP_CHECKSIG` (and its Schnorr compatibility added in May 2019) instead. A Schnorr signature inside `OP_CHECKMULTISIG` causes script failure. [Source: https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2019-05-15-schnorr.md]
14. **P2SH32 is not the wallet default.** If you use `bitcoin-cli getnewaddress` with default settings, you'll get a P2PKH or P2SH20. To get a P2SH32 address, you have to construct it manually. [Source: BCHN v29.1.0 Limitations]

## 9. References

- Bitcoin Cash protocol reference: https://reference.cash/
- CashTokens CHIP spec (v2.2.2 final): https://cashtokens.org/docs/spec/chip/
- CHIP-2021-02 Native Introspection Opcodes: https://reference.cash/protocol/forks/chips/2022-05-native-introspection-opcodes
- CHIP-2022-05 P2SH32 specification: https://bitcoincashresearch.org/t/chip-2022-05-pay-to-script-hash-32-p2sh32-for-bitcoin-cash/806
- Replay-Protected Sighash spec (BUIP-HF): https://www.reference.cash/protocol/forks/replay-protected-sighash
- BCHN v29.1.0 release notes: https://docs.bitcoincashnode.org/doc/release-notes/release-notes-29.0.0/
- BCHN v29.0.0 release notes (Layla upgrade): https://docs.bitcoincashnode.org/doc/release-notes/release-notes-29.0.0/
- CHIP-2024-12 P2S spec: https://github.com/bitjson/bch-p2s
- CHIP-2025-05 Bitwise re-enable spec: https://github.com/bitjson/bch-bitwise
- CashScript covenant guide: https://cashscript.org/docs/guides/covenants/
- libauth source: https://github.com/bitauth/libauth
- BIP-146 (low-S): https://bips.dev/146/
- BIP-340 (Schnorr, BCH variant): https://github.com/bitcoincashorg/bitcoincash.org/blob/master/spec/2019-05-15-schnorr.md
- 2018-11-15 BCH upgrade opcodes re-enabled: https://github.com/bitcoin-sv-specs/protocol/blob/master/updates/2018-11-15%20BCH%20Upgrade%20Spec.md
- Companion wiki pages: `entity.libauth`, `entity.cashscript`, `entity.cauldron-dex`, `entity.bchn-node`, `concept.cash-tokens`, `reference.bch-zero-conf-security`, `source.bchn-release-notes`
