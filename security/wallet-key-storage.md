---
pageType: reference
id: reference.security.wallet-key-storage
description: What four production BCH wallets actually do at rest — KDF, cipher, salt handling, and the platform shims that silently disable encryption. Cashonize plaintext IndexedDB, Selene AES-256-GCM via Capacitor with a passthrough web stub, Paytaca patched WebCrypto secure storage, Electron Cash ECIES/AES-CBC with PBKDF2-1024.
sourceUrl: https://github.com/cashonize/cashonize-wallet
---

# BCH wallet key storage: what production wallets actually do at rest

*Companion to [`wallet-threat-model.md`](wallet-threat-model.md), which models the threats. This page is the evidence layer: the exact KDF, cipher, salt, and file layout each production BCH wallet uses, read from source rather than from marketing. Verified 2026-10-01 against `main`/`master` HEADs (Cashonize `7ca3280`, Selene `48237e6`, Paytaca `4f84257`, Electron Cash `master`).*

The headline result: **"this wallet encrypts your seed" is a per-platform claim, not a per-wallet one.** Two of the four wallets ship a web implementation whose `encrypt()` returns its input unchanged. A wallet can be genuinely well-encrypted on iOS and genuinely plaintext in a browser tab, from the same codebase, with no error raised in between.

## 1. Summary table

| Wallet | Platform | KDF | Cipher | Salt | Encrypted by default? |
|---|---|---|---|---|---|
| **Cashonize** | web / Electron / Android | — | — | — | **No** — plaintext in IndexedDB |
| **Selene** | iOS / Android | PBKDF2-HMAC-SHA256, 100 000 iters | AES-256-GCM | 16 random bytes | **Yes** on native |
| **Selene** | web (browser) | PBKDF2 100k (PIN only) | **none — passthrough** | 16 bytes (PIN hash) | **No** — `encrypt()` is a no-op |
| **Paytaca** | iOS / Android | OS keychain / keystore | OS-managed | OS-managed | Yes |
| **Paytaca** | web / BEX | **none** | AES-256-GCM, key in IndexedDB | 12-byte IV | Encryption yes, key protection no |
| **Electron Cash** | desktop / Android | PBKDF2-HMAC-SHA512, **1024 iters, empty salt** | AES-256-**CBC** (ECIES) | **none** | Only if you set a password |

## 2. Cashonize — plaintext, and the project says so

Cashonize ships `security-considerations.md` at the repo root, which is unusually honest and worth reading before any external claim:

> "Cashonize currently stores your seed phrase unencrypted in IndexedDB." — `cashonize-wallet/security-considerations.md`

It goes on to state the threat model per platform: the webwallet and installable web app expose the seed to "any browser extension with access to that origin"; the Electron desktop build stores it in plaintext on disk; the Android `.apk` gets per-app sandboxing "which is a meaningful default protection that the desktop and web variants do not have."

Encrypted storage is described as future work: *"Encrypted seed storage is on the roadmap."*

**Verified in code.** A repo-wide search for encryption in the seed path returns nothing. The only KDF in the entire wallet is for BIP-38 key import:

```ts
// cashonize-wallet/src/utils/tools/bip38.ts:15-18
const passphraseScryptParams = { N: 16384, r: 8, p: 8 };   // BIP-38 mandated
const passpointScryptParams = { N: 1024, r: 1, p: 1 };     // BIP-38 mandated
```

Those parameters are fixed by [BIP-38](https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki) and apply to importing a paper-wallet private key, never to the wallet's own seed. Searching for `encrypt` across `src/` and `test/` matches only `bip38.ts` and the sweep-private-key component. There is no encrypted-at-rest path for the seed.

**Entropy sourcing is solid, though.** Seed generation is `libauth`'s `generateBip39Mnemonic` at 128 bits, drawing on `crypto.getRandomValues`; `security-considerations.md` documents the OS-level backing (Linux/Android `getrandom()`, Windows `ProcessPrng`, macOS Fortuna) and notes libauth runs defensive checks against broken RNG implementations. The weakness is storage, not generation.

**Supply-chain posture is the strongest of the four.** `package.json` pins exact versions rather than ranges — `"@bitauth/libauth": "3.1.0-next.8"`, `"mainnet-js": "3.1.7"`, `"@mainnet-cash/indexeddb-storage": "3.1.7"`, `"@noble/ciphers": "2.4.0"`, `"@noble/hashes": "2.4.0"`, plus `"packageManager": "pnpm@11.10.0"` and a committed `pnpm-lock.yaml`. CI (`.github/workflows/ci.yml`) runs `pnpm install --frozen-lockfile`, lint, `vue-tsc --noEmit`, an i18n check, and `vitest`. `SECURITY.md` routes reports to GitHub private vulnerability reporting, and — notably — **explicitly declares the plaintext storage and web-delivery model out of scope**: *"Out of scope are the known trade-offs documented in the security considerations ... in particular the plaintext seed storage and the code delivery model of the webwallet."*

One CI caveat worth recording: `pnpm test:e2e` is `continue-on-error: true`, with a comment that the CashConnect and WalletConnect suites time out against public relays. A wallet-integration e2e suite that is not gating is not evidence of a working integration.

## 3. Selene — real encryption on native, a no-op on web

Selene delegates all key handling to `capacitor-plugin-simple-encryption` (pinned to a git tag: `https://git.xulu.tech/selene.cash/capacitor-plugin-simple-encryption.git#v0.2.9`). The kernel service is `src/kernel/app/SecurityService.ts`; the storage layer is `src/kernel/app/DatabaseService.ts`.

**Architecture.** A random 256-bit data-encryption key (DEK) is generated once and used to encrypt SQLite database blobs and exported wallet files. The DEK is then wrapped by a PIN-derived key and stored in platform secure storage. This is the correct construction — the expensive KDF runs once at unlock, not per database.

**KDF, from the native sources:**

- iOS — `ios/Sources/SimpleEncryptionPlugin/KeyManager.swift:15`: `private static let pbkdf2Iterations: UInt32 = 100_000`, used via CommonCrypto `CCPBKDFAlgorithm(kCCPBKDF2)` at line 481, salt `pinSaltSize = 16` bytes (line 13). Wrapped blob layout is `[salt][nonce ‖ ciphertext ‖ tag]`, `encryptedKeySize = 12 + 32 + 16` (line 16).
- Android — `android/.../KeyManager.kt:41`: `private const val PBKDF2_ITERATIONS = 100_000`, `deriveKeyPbkdf2(newPin, salt, PBKDF2_ITERATIONS)` at line 191, salt `generateRandomBytes(16)`. Cipher is `AES/GCM/NoPadding` (line 387).

**Keychain/Keystore properties are the strong part.** iOS uses `SecAccessControlCreateWithFlags` with `.biometryCurrentSet` and `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` (`KeyManager.swift:240-243`). `.biometryCurrentSet` means adding a new fingerprint or face to the device **invalidates the stored key** — the right choice, and the one most wallets get wrong. Android uses `KeyGenParameterSpec` with `setUserAuthenticationRequired(true)`, `setInvalidatedByBiometricEnrollment(true)`, and `setUserAuthenticationParameters(0, AUTH_BIOMETRIC_STRONG)` (`KeyManager.kt:357-369`) — auth-per-use with a zero-second window, so the key is usable while the session is unlocked. `setDeviceOnlyMode(true)` opts out of cloud backup entirely, and the code comments on Android Auto Backup explicitly: *"Auto Backup restores SharedPreferences but not Android Keystore keys"* — the failure mode where a restored backup leaves unwrappable ciphertext is handled deliberately (`KeyManager.kt:478`).

**Brute-force resistance.** Android keeps an escalating lockout ladder with attempts persisted in `SharedPreferences` under `SimpleEncryptionLockout`, capped at 8 attempts and `MAX_LOCKOUT_SECONDS = 3600` (`KeyManager.kt:42-45, 855-892`). A correct PIN resets the counter. Note the ceiling: an attacker with the extracted blob can make at most 8 attempts per hour per device before the wall, and there is no server-side component, so this only slows offline attack on the device itself.

### 3.1 The finding that matters: the web implementation is a passthrough

```ts
// capacitor-plugin-simple-encryption/src/web.ts:102-109
async encrypt(options: EncryptOptions): Promise<EncryptResult> {
  if (localStorage.getItem(PIN_HASH_KEY) !== null && !isUnlocked) {
    throw new Error("Encryption key is locked. Call unlockWithPin() first");
  }
  // Passthrough - return data unchanged
  return { data: options.data };
}
```

The plugin's own header comment is explicit: *"Web browsers cannot securely store encryption keys, so we return data unchanged. The app should warn users that web storage is not encrypted."* On `initialize()` it logs `"[SimpleEncryption] Running on web - encryption not available. Data will be stored unencrypted."`

This is not merely theoretical for Selene, because the consumer code branches on it. In `WalletManagerService.ts` the export path always calls `SimpleEncryption.encrypt(...)` before writing the wallet file (`src/kernel/wallet/WalletManagerService.ts:491`), and the import path sniffs the format rather than trusting the platform:

```ts
// src/kernel/wallet/WalletManagerService.ts:514-519
// Decrypt if encrypted (not plain JSON — web stub is a passthrough)
if (!fileData.startsWith("{")) {
  ({ data: fileData } = await SimpleEncryption.decrypt({ data: fileData }));
}
```

And `reencryptAllData` short-circuits entirely off-native (`DatabaseService.ts:380-382`):

```ts
async function reencryptAllData(oldKeyBase64: string): Promise<string[]> {
  if (!Capacitor.isNativePlatform()) {
    return [];
  }
```

So a wallet file exported from a browser build is plain JSON containing `mnemonic` and `passphrase` (`WalletManagerService.ts:480-489`), and moving it to a phone will not be detected or warned about by any code path beyond the `{` sniff. **When evaluating a Selene wallet file, check whether it starts with `{` before assuming it is encrypted.**

The PIN gate on web is also weak in a specific way. The web shim still enforces a lock (`encrypt()` throws while locked), but `encrypt()` still returns the input, so the PIN gates *UI access to the app*, not the confidentiality of the data at rest. PIN verification itself is PBKDF2-SHA256 at 100 000 iterations against a `salt$hash` pair in `localStorage` (`web.ts:163-207`), with a constant-time comparison loop — correct for what it is, but it protects a PIN, not a seed.

**Memory hygiene, honestly labelled.** `KeyManagerService.ts:33-42` caches derived seeds in a `Map` and zeroes them on lock:

```ts
// Note: Map.clear() releases references but does NOT zero the seed bytes in
// V8 heap memory. JavaScript provides no reliable way to overwrite GC-managed
// memory. On native platforms, key material is zeroed via memset_s (iOS) and
// ByteArray.fill(0) (Android). This JS-side clear is best-effort only.
export function clearSeedCache() { ... }
```

Selene is the only wallet in this set whose source comments acknowledge the GC limitation rather than implying the bytes are gone. `SecurityService.lock()` and `securePause()` both call `clearSeedCache()`, close all database handles, and call `SimpleEncryption.clearKeyFromMemory()`.

### 3.2 Selene has no desktop build

The wiki previously described Selene as shipping "on desktop (Electron) and mobile (iOS/Android via Capacitor)". **This is stale.** Verified: `package.json` has no Electron dependency, the repo has no `src-electron/` directory, and the only occurrences of the string "electron" anywhere in the tree are two commented-out Electrum server hostnames in `src/util/network.ts` (`electroncash.dk:50004`, `electroncash.de:55004`). The README lists the tech stack as React + Vite + Capacitor 8 + Redux + Tailwind + sql.js, and gives build instructions for Android and iOS only. `selene.cash` offers "Try Web Version" and "Download .apk" (with a published sha256) — no desktop installer. The `electroncash.org` results that surface alongside Selene in search are Electron Cash's own apps, a different wallet.

The practical consequence: the "Selene is weaker on desktop" caveat has no desktop to apply to. Selene is native-or-web, and the web path is the passthrough.

## 4. Paytaca — a patch is the security story

Paytaca stores mnemonics through `capacitor-secure-storage-plugin` under a per-wallet key:

```js
// paytaca-app/src/wallet/index.js:164-175
export async function getMnemonicByHash(walletHash) {
  const key = `mn_${walletHash}`
  try {
    const mnemonic = await SecureStoragePlugin.get({ key })
    return mnemonic.value
  } catch (err) {
    // Propagate decryption failures so callers can distinguish
    // "key lost/corrupted" from "not found"
    if (isSecureStorageDecryptError(err)) throw err
    return null
  }
}
```

That error-discrimination discipline is worth noting: Paytaca explicitly distinguishes "the value is gone" from "the value is there but I cannot decrypt it", rather than collapsing both to `null`.

**The upstream web implementation stores values as plain base64 in `localStorage`.** Paytaca fixes this with a `patch-package` patch shipped in the repo — `patches/capacitor-secure-storage-plugin+0.5.1.patch`:

> *"Paytaca patch: encrypt values at rest on web/BEX (AES-256-GCM). The upstream web implementation stored values as plain base64 in localStorage. This patch encrypts values with a non-extractable AES-GCM CryptoKey kept in IndexedDB, so raw key material is never exposed to JS. Legacy base64 values are still readable and are lazily re-encrypted."*

The patch generates a non-extractable `crypto.subtle` AES-GCM-256 key (`generateKey({ name: 'AES-GCM', length: 256 }, false, ['encrypt','decrypt'])` — the `false` is `extractable`), stores it in an IndexedDB object store `paytaca-secure-storage`, and prefixes ciphertext with `enc:v1:`. Legacy values are read once and re-encrypted lazily. Failure to decrypt throws the sentinel `SECURE_STORAGE_DECRYPT_FAILED`, which is exactly what `isSecureStorageDecryptError` matches.

**What this does and does not buy you.** The threat it blocks is real: an extension with origin access, or another script on the same origin, can no longer read the mnemonic out of `localStorage`. What it does not provide is protection from anything with access to the origin's IndexedDB itself — the AES key sits in IndexedDB next to the ciphertext, and both are reachable to any script running on `cashonize`-like origins under the same profile. The design is "not readable by a careless extension", not "not readable by code execution on the origin". For a browser build, that is a meaningful improvement over base64 and a modest one over no protection at all.

Paytaca also runs a legacy-migration path for pre-v0.9.1 wallets that used a different scheme (`src/wallet/mnemonic-migration.js`, `aes256.decrypt(...)` against a `SecureStoragePlugin` value), and pins several security-relevant dependencies to exact versions including `@bitauth/libauth: 2.0.0-alpha.8` — an **alpha** crypto library on a production wallet holding real funds, and a dependency worth watching. `@cashlab/cauldron: ^1.0.2` and `@cashscript/utils: ^0.11.5` are semver ranges.

## 5. Electron Cash — the incumbent's cipher is older than it looks

Electron Cash is the reference BCH desktop wallet and, as of this writing, the most widely deployed one (latest releases 4.4.6 / 4.4.6.1, Sept 2026). Its wallet-file encryption is **opt-in** — `create_new_wallet` takes `encrypt_file=True` by default but a null/empty password still produces a plaintext file, and `set_password` only engages the cipher when `encrypt and password` are both truthy.

The full chain, read from source:

```python
# Electron-Cash/electroncash/storage.py::get_key
secret = hashlib.pbkdf2_hmac('sha512', password.encode('utf-8'), b'', iterations=1024)
ec_key = bitcoin.EC_KEY(secret)
```

```python
# Electron-Cash/electroncash/storage.py::_write
s = zlib.compress(s)
s = bitcoin.encrypt_message(c, self.pubkey)
```

**KDF: PBKDF2-HMAC-SHA512, 1024 iterations, empty salt.** The salt argument is literally `b''`. 1024 iterations of PBKDF2-SHA512 is a 2013-era default — on modern hardware this is a sub-millisecond derivation, which makes an offline attack on a stolen wallet file a matter of raw hashing throughput rather than a meaningful cost. This is inherited from Electrum's original design and has been unchanged across Electron Cash's history; it is not a defect introduced by Electron Cash, but it is the weakest link in the most widely used BCH wallet's at-rest protection.

**Cipher: ECIES-style, AES-256-CBC with HMAC-SHA256.**

```python
# Electron-Cash/electroncash/bitcoin.py::EC_KEY.encrypt_message
ephemeral_exponent = ecdsa.util.randrange(pow(2, 256), generator_secp256k1.order())
ephemeral = EC_KEY(ephemeral_exponent)
ecdh_key = point_to_ser(pk * ephemeral.privkey.secret_multiplier)
key = hashlib.sha512(ecdh_key).digest()
iv, key_e, key_m = key[0:16], key[16:32], key[32:]
ciphertext = aes_encrypt_with_iv(key_e, iv, message)
encrypted = b'BIE1' + ephemeral_pubkey + ciphertext
mac = hmac.new(key_m, encrypted, hashlib.sha256).digest()
```

ECDH over secp256k1 derives a 64-byte material split into IV / encryption key / MAC key; the plaintext is zlib-compressed first, then AES-256-**CBC** (`AES.new(key, AES.MODE_CBC, iv)`) with PKCS7 padding, and authenticated by a SHA-256 HMAC over the ciphertext. Decryption is encrypt-then-MAC with the MAC verified before any decryption, and a MAC failure raises `InvalidPassword`.

So Electron Cash is **authenticated** — no unauthenticated-ciphertext malleability — but **not AEAD**, and the padding oracle surface is handled by raising `InvalidPassword` on bad padding. The practical read: this is a sound 2013-era construction, not a broken one. The KDF cost is the problem, not the cipher.

Wallet-file handling is otherwise disciplined: writes go to a temp path, are `fsync`'d, then `os.replace`d into position (`storage.py::_write`), and the file is created with mode `S_IREAD | S_IWRITE` (0600). Seed and passphrase are separately encrypted via `pw_encode`/`pw_decode` inside the keystore, and password validation checks structural validity rather than only a MAC — `check_password` decodes the xprv and compares the derived chain code against the stored xpub's (`keystore.py::Xprv.check_password`), so a wrong password cannot silently produce a valid-looking key.

Historical security items in Electron Cash's own release notes, cited rather than characterised: "Fixes JSONRPC security threat" and "Additional security improvements related to JSONRPC" (3.1.1, 3.1.2); "Android only release for security fixes" (3.1.3); "Due to phishing being a recurrent annoyance, 'Preferred servers only' now defaults to CHECKED for new installs"; later "Made 'Connect only to Preferred Servers' False by default" (4.0.15) — i.e. the phishing mitigation was later reversed, so do not rely on it; "Hardened build system against dependency vulnerabilities" (3.4.x); a CashShuffle bug where *"some shuffle tx's use the same address twice in the outputs"* (4.0.14, a de-anonymisation bug, patched).

## 6. Cross-cutting lessons

1. **Check the platform shim, not the marketing page.** Two wallets here ship a web `encrypt()` that returns its input unchanged. A security review that reads only the native code path will conclude both wallets are well encrypted.
2. **Format-sniffing beats platform-checking.** Selene's `if (!fileData.startsWith("{"))` decrypt-or-not heuristic is the correct pattern: it is true regardless of which platform wrote the file. A platform boolean is only true for the platform that wrote it.
3. **`.biometryCurrentSet` / `setInvalidatedByBiometricEnrollment(true)` is the correct default**, and Selene's plugin gets it right on both platforms. Wallet file backups that survive but keys that don't are a common and confusing failure, and Android Auto Backup + Keystore is exactly that case — which the plugin explicitly documents and handles.
4. **PBKDF2 iteration counts are the thing to check when reading any wallet's KDF.** None of these wallets uses scrypt or Argon2id for wallet encryption. Electron Cash's 1024 is the outlier in the bad direction; Selene's 100 000 PBKDF2-SHA256 and BIP-38's mandated N=16384 scrypt are defensible; the ecosystem's most common choice, scrypt, appears only because bch-bot opted into it.
5. **Patches in the repo are security-relevant code.** Paytaca's entire web-side encryption story lives in `patches/`, invisible to anyone reading `package.json` dependencies.
6. **Zeroisation claims deserve scepticism in every runtime.** Selene is to be credited for saying so in a comment rather than implying `fill(0)` in a JS heap actually scrubs memory.

## What's not solved

- **No production BCH wallet offers plausible deniability for the seed itself.** All four store or derive from a recoverable mnemonic. A BIP-39 passphrase that is prompted for at runtime and never persisted is the only mechanism in this set that gets close, and none of the four wallets prompt for it separately from wallet unlock.
- **Browser-based BCH wallets remain the weakest surface by construction.** A web wallet is served live, executes in a browser with extension access, and — for two of the four — stores the seed unencrypted. Cashonize documents this candidly rather than pretending otherwise, which is the correct posture, but it does not change the underlying property.
- **No audit of any of these wallets is public.** Cashonize has a `SECURITY.md` and a private reporting channel; the others have none found. Nobody in this set has published a third-party security review.
- **Electron Cash's KDF has not been re-hardened.** Fixing the iteration count breaks every existing wallet file's decryptability unless a migration path is built, which is presumably why nobody has done it. It remains the most consequential single-line finding on this page.
- **CashTokens keys are not separately protected anywhere.** Every finding here applies equally to the token-bearing signing path; none of the four wallets isolates covenant/token capability keys from ordinary spend keys at rest.

## Sources

- Cashonize: <https://github.com/cashonize/cashonize-wallet> — `security-considerations.md`, `SECURITY.md`, `package.json`, `.github/workflows/ci.yml`, `src/utils/tools/bip38.ts`; HEAD `7ca3280`
- Selene Wallet: <https://gitlab.com/selene.cash/selene-wallet> — `src/kernel/app/SecurityService.ts`, `src/kernel/app/DatabaseService.ts`, `src/kernel/wallet/WalletManagerService.ts`, `src/kernel/wallet/KeyManagerService.ts`, `package.json`, `README.md`; HEAD `48237e6`
- Selene encryption plugin: <https://git.xulu.tech/selene.cash/capacitor-plugin-simple-encryption> — `src/web.ts`, `ios/Sources/SimpleEncryptionPlugin/KeyManager.swift`, `android/src/main/java/cash/selene/plugins/simpleencryption/KeyManager.kt`
- Paytaca: <https://github.com/paytaca/paytaca-app> — `src/wallet/index.js`, `src/wallet/mnemonic-migration.js`, `patches/capacitor-secure-storage-plugin+0.5.1.patch`, `package.json`; HEAD `4f84257`
- Electron Cash: <https://github.com/Electron-Cash/Electron-Cash> — `electroncash/storage.py`, `electroncash/bitcoin.py`, `electroncash/keystore.py`, `RELEASE-NOTES`; releases 4.4.6 / 4.4.6.1
- Selene distribution: <https://selene.cash/> — Web + `.apk` only, no desktop build
- BIP-38: <https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki>
