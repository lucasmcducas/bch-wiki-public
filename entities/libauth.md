
---
pageType: entity
entityType: library
id: entity.libauth
description: BitAuth libauth TypeScript library for BCH transaction construction, signing, and address derivation. Powers CashTokens-aware signing for the bot.
sourceUrl: https://github.com/bitauth/libauth
---

# libauth

> BitAuth libauth — TypeScript library for BCH transaction construction, signing, and address derivation. Powers CashTokens-aware signing for the bot.

**Repository:** [github.com/bitauth/libauth](https://github.com/bitauth/libauth)
**Last verified:** 2026-09-17
**Sources:** README, CHANGELOG, `src/lib/` tree on master (v3.1.0-next.8 / HEAD)

## What libauth is

An ultra-lightweight JavaScript/TypeScript library maintained by BitAuth (Calin Culianu / bitjson). Three big things: BCH/BTC/Bitauth transaction **parsing and construction**, **address derivation and encoding**, and a **wallet-template compiler** that turns human-readable `auth-template.json` files into BCH scripts. It also ships a BCH virtual machine that can execute covenant scripts offline (no node connection required).

**Current version** is **v3.1.0-next.8** on master; v3.0.0 is the latest released version. CashTokens support was added in **v2.0.0** (commit `8e99139accb973f1df82b4cbcc92eeb81af77e0c`, May 2023) — same release that introduced `signing_serialization.token_prefix`. The `token_prefix` signing-serialization component was retrofitted/clarified in **v3.0.0** (PR #127, commit `e5c275f`); BCH-capable wallets and the compiler wired it up correctly there.

## The architecture you must understand

**Libauth has no standalone `signTransaction`.** This is the most important architectural fact. Calling `signTransaction(tx, key, ...)` will not work; the function does not exist. Instead, libauth decomposes signing into small composable pieces:

> **2026-09-17 update:** This is true for ad-hoc signing, BUT libauth v3 ships a **wallet-template compiler** that wraps the manual path. For P2PKH-only sends, `importWalletTemplate(walletTemplateP2pkhNonHd)` + `walletTemplateToCompilerBCH` lets you sign in ~30 LOC without touching `encodeSigningSerializationBCH`. Selene Wallet's `KeyManagerService.signInputs` is the canonical example. The manual path below is still required for CashScript contract inputs. See `synthesis.bch-wallet-bot-build-log.md` for the build experience.

1. Build the unsigned transaction object.
2. For each input, determine which sighash algorithm applies (BCH-specific flag set, see below).
3. Compose the sighash preimage using `encodeSigningSerializationBCH({ ..., coveredBytecode, outputTokenPrefix, ... })`. The `outputTokenPrefix` is `encodeTokenPrefix(sourceOutputs[inputIndex].token)` — and **this is what makes signing CashTokens-aware**.
4. Hash the preimage with `hash256` (double-SHA256).
5. Sign the resulting 32-byte hash by calling `secp256k1.signMessageHashSchnorr(...)` or `secp256k1.signMessageHashDER(...)` on the consumer-supplied Secp256k1 interface (typically a WASM binding).
6. Append the appropriate sighash byte to the signature, build the unlocking bytecode (typically `<push sig> <push pubkey>`, or for P2SH the redeem script + sig).
7. Set `input.unlockingBytecode` and finalize.

A "signing entry point" exists in the **compiler path** (`compiler-bch.ts`) where the Bitauth IDE-style template compiler wires this up — but it is *not* a public function you can call for an ad-hoc libauth-built transaction. You compose the pipeline yourself.

## Version 3 surface: signing-serialization flags and SIGHASH bytes

The sighash type byte is computed by `OR`-ing flags from `src/lib/vm/instruction-sets/common/signing-serialization.ts`:

| Flag             | Byte     | Name             |
|------------------|----------|------------------|
| `allOutputs`     | `0x01`   | SIGHASH_ALL      |
| `noOutputs`      | `0x02`   | SIGHASH_NONE     |
| `correspondingOutput` | `0x03`| SIGHASH_SINGLE |
| `utxos`          | `0x20`   | SIGHASH_UTXOS (CashTokens, mandatory for token inputs) |
| `forkId`         | `0x40`   | BCH replay protection |
| `singleInput`    | `0x80`   | ANYONECANPAY     |

Pre-computed one-byte combinations (full byte, what you append to signatures):

| Algorithm name            | Byte     | Meaning                           |
|---------------------------|----------|-----------------------------------|
| `allOutputs`              | `0x41`   | SIGHASH_ALL \| SIGHASH_FORKID     |
| `allOutputsAllUtxos`      | `0x61`   | + SIGHASH_UTXOS (CashTokens)      |
| `allOutputsSingleInput`   | `0xC1`   | + ANYONECANPAY                    |
| `allOutputsSingleInputAllUtxos` | `0xE1` | both                             |
| `noOutputs`               | `0x42`   | SIGHASH_NONE \| SIGHASH_FORKID    |
| `noOutputsAllUtxos`       | `0x62`   | + SIGHASH_UTXOS                   |
| `correspondingOutput`     | `0x43`   | SIGHASH_SINGLE \| SIGHASH_FORKID  |
| `correspondingOutputAllUtxos` | `0x63` | + SIGHASH_UTXOS                |
| `correspondingOutputSingleInput` | `0xC3`| SIGHASH_SINGLE+ANYONECANPAY      |
| `correspondingOutputSingleInputAllUtxos` | `0xE3` | all four        |

The most common prefix byte you'll see on a sig in the wild is `0x41` (SIGHASH_ALL | SIGHASH_FORKID). For BCH today, leave the `forkId` parameter at the default zero bytes; forkId is **3 bytes**, and using a non-zero forkId requires appending those bytes to the sighash prefix.

`SIGHASH_UTXOS` (`0x20`) is mandatory for any input that spends a token-bearing UTXO — without it, the consensus rule rejects the signature. The signing preimage in that case includes a `hashUtxos` field (`hash256(encodeTransactionOutputsForSigning([utxoBeingSpent]))`).

## Sign with a Secp256k1 backend

You bring your own Secp256k1 implementation; libauth consumes it through a `Secp256k1` interface. The two functions the wallet calls into:

- `secp256k1.signMessageHashDER(hash, privateKey)` — ECDSA, returns DER-encoded `r||s` (variable length).
- `secp256k1.signMessageHashSchnorr(hash, privateKey)` — BIP-340 Schnorr, returns fixed 64-byte `r||s`.

Libauth does not implement secp256k1 itself; `libauth-template` and `mainnet-js` typically use the WASM `libsecp256k1` binding or `@noble/secp256k1`. The moth bch-wallet skill uses `@bitauth/libauth` with a separate secp256k1 binding.

## Key & address primitives

From `src/lib/key/`:

- `deriveSeedFromBip39Mnemonic(mnemonic, passphrase?)` — returns 64-byte seed.
- `decodeHdPrivateKey(extendedKey)` / `encodeHdPrivateKey(...)` — BIP32 xprv handling.
- `deriveHdPathRelative(seedOrXpriv, pathLikeString)` — supports **relative BIP32 derivation** added in v3.0.0.
- `deriveHdPublicKey(publicNode, derivationPath)` — derives a child pubkey from a parent xpub.
- `bip32HmacSha512Key(...)` — internal.

From `src/lib/address/cash-address.ts`:

- `encodeCashAddress({ payload, prefix, version })` / `decodeCashAddress(address)` — the canonical cashaddr encoding.
- `encodeCashAddressFormat / decodeCashAddressFormat` — same but auto-strips the prefix.
- `lockingBytecodeToCashAddress({ bytecode, prefix? })` / `cashAddressToLockingBytecode(address)` — inverse mapping.
- `CashAddressType` enum: `p2pkh | p2sh20 | p2sh32` (CashTokens token-aware addresses, type-byte 3).

From `src/lib/address/locking-bytecode.ts`:

- `encodeLockingBytecodeP2pkh(hash160)` — P2PKH lock.
- `encodeLockingBytecodeP2sh20(hash160)` — legacy P2SH (20-byte).
- `encodeLockingBytecodeP2sh32(hash256)` — P2SH32 (32-byte, CashTokens-aware covenant locks).
- `decodeLockingBytecode(...)` — disassembles back to opcodes.
- `lockingBytecodeLength(...)` — length helper.

Also useful: `encodeDataPush(data: Uint8Array)`, `OP_RETURN` (`0x6a`) push for OP_RETURN outputs (`flattenBinArray([hex('6a'), ...encodeDataPush(chunk1), ...encodeDataPush(chunk2), ...])`).

**BIP44 path for BCH** is `m/44'/145'/0'/0/i` (external/receive) — verified in `src/lib/compiler/p2pkh-utils.ts` L167-168. Coin type **145** is SLIP-0044 for Bitcoin Cash. Change chain is `m/44'/145'/0'/1/i`. Address index `i` starts at 0.

WIF: `encodePrivateKeyWif({ privateKey, prefix })` / `decodePrivateKeyWif(wif)`. Validation: `validateSecp256k1PrivateKey(privateKey)` — checks `< n`.

## Transaction output shape (CashTokens)

A BCH output carrying tokens in libauth v3:

```ts
const ftOut: Output = {
  lockingBytecode: ...,        // P2PKH or covenant
  valueSatoshis: 1000n,        // bigint; 0 if pure NFT
  token: {
    amount: 100n,              // bigint, fungible amount
    category: categoryBytes,   // 32-byte big-endian category id
    // nft undefined → fungible only
  },
};

const nftOut: Output = {
  lockingBytecode: ...,
  valueSatoshis: 0n,
  token: {
    amount: 0n,
    category: categoryBytes,
    nft: {
      capability: 'mutable',   // 'none' | 'mutable' | 'minting'
      commitment: commitmentBytes,
    },
  },
};
```

`encodeTokenPrefix(token)` produces the byte string that goes into the signing preimage's `outputTokenPrefix` field — `0xef || VarInt(category) || amount || (nft.capability, nft.commitment)`. The capability numbers are 0/`none`, 1/`mutable`, 2/`minting`.

## Wallet-template compiler

`src/lib/compiler/compiler-bch/compiler-bch.ts` is the BCH variant of the template compiler. The relevant export is:

- `compileTemplate(...)` — returns a `CompilationResult` with the witness scripts and sighash generators.
- `compilerOperationSigningSerializationFullBCH` (line 608) — handles any of the 13 algorithm names and produces the full preimage.
- `compilerOperationSigningSerializationTokenPrefix` (line 576) — the standalone `signing_serialization.token_prefix` component. Uses `encodeTokenPrefix(sourceOutputs[inputIndex].token)` — this is how CashTokens-aware sighash is wired into the compiler.

If you author an auth-template JSON in the Bitauth IDE, this compiler handles everything for you. If you want to bypass the compiler and build a tx from primitives (which is what most wallet bots do for ad-hoc payments), use the lower-level signing-serialization exports directly.

## References to read while implementing

| Export                                                | Path                                                                  |
|-------------------------------------------------------|-----------------------------------------------------------------------|
| `encodeSigningSerializationBCH`                       | `src/lib/vm/instruction-sets/common/signing-serialization.ts` L221-346 |
| `generateSigningSerializationComponentsBCH`           | same file L440-476                                                   |
| `hashPrevouts / hashUtxos / hashSequence / hashOutputs`| same file L109-219                                                   |
| `encodeTransactionInputsForSigning`                   | `src/lib/message/transaction-encoding.ts` L820-833                    |
| `encodeTransactionInputSequenceNumbersForSigning`     | same file L835-837                                                   |
| `encodeTokenPrefix`                                   | `src/lib/message/transaction-encoding.ts` L170-228 (CashTokens errors)|
| `SigningSerializationFlag` / `SigningSerializationType` (enums) | `src/lib/vm/instruction-sets/common/signing-serialization.ts` L20-74 |
| `compiler-bch.ts` (BCH compiler)                      | `src/lib/compiler/compiler-bch/compiler-bch.ts` L1-759                |

## Pitfalls

1. **There is no `signTransaction`.** Compose `encodeSigningSerializationBCH` + consumer-side secp256k1 sign. The compiler path hides this — most Stack Overflow answers will mislead you into looking for a function that doesn't exist.
2. **P2S (CHIP-2024-12) is NOT yet exposed in libauth.** A `grep` for `p2s`, `P2STATIC`, `chip-2024-12` finds zero matches in the master tree (as of v3.1.0-next.8). If you need P2S locking-bytecode helpers, you write the script bytes yourself or wait.
3. **The legacy `deriveHdPath` is deprecated.** Use the relative-derivation API added in v3.0.0 (PR #127). The full-derivation variants still work but emit deprecation noise in dev builds.
4. **The `signing_serialization.token_prefix` field is mandatory for token inputs.** Forgetting it means a valid-looking sighash that consensus rejects. The compiler wires it for you; manual transaction construction must include it explicitly.
5. **The compiler operation `.signature` was renamed in v3 to `.ecdsa_signature` / `.schnorr_signature`.** Older auth-template files referring to `.signature` against `Secp256k1` will fail to compile.
6. **For P2SH inputs, the covered script is the redeem script, NOT the script hash.** Get this wrong and you'll sign a preimage that never validates on chain.
7. **`SIGHASH_SINGLE` (0x03) has a SIGHASH_FORKID (0x40) variant.** Make sure you compute the byte correctly — `0x43`, not `0x03`.
8. **`Secp256k1` no longer throws in v3.** Errors are returned as `string`. Don't wrap calls in try/catch expecting throws.
9. **Pure ESM only.** `import { ... } from '@bitauth/libauth'` — no CommonJS path. Downstream projects need `"type": "module"` or proper transpilation.
10. **No internal Schnorr or signing function exists in libauth.** It does parsing, hashing, serialization, and address work; it deliberately delegates signing to a secp256k1 implementation you supply. Tooling that *uses* libauth for signing must own a Secp256k1 implementation (WASM or pure-JS).

## Related entities

- `entity.bchn-node` — the full node libauth talks to indirectly through Fulcrum/indexer layer.
- `entity.cashscript` — higher-level compiler that emits cashc → CashScript artifacts; libauth handles the runtime types/signatures those artifacts reference.
- `entity.moth-bch-wallet` — a real-world Node.js wallet built on `@bitauth/libauth` + `@electrum-cash/protocol`, the moth pattern this bot aims to replicate.
- `concept.cash-tokens` — the FT/NFT shape this library's signing serialization depends on.
