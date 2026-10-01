---
pageType: reference
id: reference.security.wallet-threat-model
description: BCH wallet security reference covering BIP-39/BIP-32 derivation, m/44'/145'/x' paths, change-chain privacy, key-at-rest, npm supply-chain threats, and counter-measures, with reference to the actual bch-bot and Selene code paths.
sourceUrl: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
---

# BCH wallet threat model

> Bitcoin Cash wallet security — from BIP-39 mnemonic generation through address derivation, key storage, and operational threats (compromised host, malicious npm dependencies, supply-chain attacks on libauth/cauldron). Anchored to the **actual** bch-bot code paths at `/home/luke/bch-bot/lib/wallet.mjs` and `/home/luke/bch-bot/lib/sign.mjs` and the Selene kernel that the bot mirrors.

## 1. BIP-39 mnemonic generation

`lib/wallet.mjs::createWallet` calls `generateBip39Mnemonic()` from `@bitauth/libauth`. BIP-39 ([spec](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)) defines:

1. **Entropy source.** The spec requires 128–256 bits of "initial entropy" but does not mandate a CSPRNG. Modern implementations use `crypto.getRandomValues` (browser), `crypto.randomBytes` (Node), or `/dev/urandom` — all CSPRNGs. libauth uses `crypto.getRandomValues`. Wallet authors must verify the runtime actually provides one; a browser without it would silently degrade.
2. **Checksum.** `CS = ENT/32` bits appended from `SHA256(entropy)`. 128-bit entropy (12-word sentence) yields 132 bits split into 12 × 11-bit wordlist indices. Word typos caught by checksum **unless** the typo lands on another valid word with the same leading bits; first-4-letter-uniqueness is the second defence.
3. **Wordlist.** BIP-39 "strongly discourages" non-English wordlists — seed derivation hashes the literal NFKD-normalized bytes; translating between wordlists produces a **different seed**. The bch-bot assumes English.
4. **Mnemonic → seed.** `PBKDF2-HMAC-SHA512(salt="mnemonic"+optional_user_passphrase, iter=2048, dkLen=64)`. The passphrase is **not stored anywhere** — BIP-39's plausible-deniability property. **The bch-bot accepts a `passphrase` arg**, which is the BIP-39 passphrase (not the wallet-encryption passphrase) and is written into `wallet.json` as part of the wallet record. When no wallet passphrase is supplied the file is written in the legacy plaintext form (`version: 1`); supplying one produces `version: 2`, scrypt + AES-256-GCM at rest, and the BIP-39 passphrase is then only ever inside the ciphertext. **The passphrase is never stored alongside a v2 wallet** — only the KDF salt and cipher IV/tag. See [§6](#6-key-storage).

> **Correction (2026-10-01).** This page previously stated the passphrase was always stored in plaintext. That was true when written and is now fixed in the code: `createWallet({ walletPassphrase })` encrypts. The plaintext path still exists as the default when no passphrase is given, so the caveat is now about *default configuration* rather than a missing feature.

**Import validation.** libauth's `deriveSeedFromBip39Mnemonic` returns a `string` error on invalid checksum. The bch-bot's `loadHdNode()` checks `typeof hdNode === 'string'` only after HD-node construction, so a failed checksum throws deep inside derivation with a confusing error. **Mitigation: pre-validate the mnemonic before calling HD.**

## 2. BIP-32 HD derivation

The seed from §1 feeds `deriveHdPrivateNodeFromSeed(seed, { assumeValidity: true })`. The master node is `(k_master, c_master)` — private key + 32-byte chain code. BIP-32 ([spec](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki)) defines:

- **Non-hardened child** (`i < 2³¹`): `I = HMAC-SHA512(c_par, ser_P(K_par) || ser_32(i))`. Parent **public key** is mixed in. Watch-only wallets can derive these from `xpub`.
- **Hardened child** (`i ≥ 2³¹`): `I = HMAC-SHA512(c_par, 0x00 || ser_256(k_par) || ser_32(i))`. Parent **private key** is mixed in. Not derivable from `xpub` alone.

**Why BCH uses hardened levels 0–2.** The bch-bot ([wallet.mjs L91–98](/home/luke/bch-bot/lib/wallet.mjs)) derives:

```
path = [44 + HARDENED, 145 + HARDENED, account + HARDENED, change, index]
       m/44'         m/145'           m/account'       /change /index
```

Purpose `44'` hardened (else an attacker with your `xpub` derives accounts at all purposes); coin type `145'` hardened (else a BTC `xpub` derives BCH addresses); account `a'` hardened (else one account's `xpub` reaches others). **Levels 3 (change) and 4 (index) are non-hardened by design** — what a watch-only wallet enumerates. libauth's `deriveHdPrivateNodeChild(node, step)` returns `HdPrivateNodeValid` or an error string when `(parse256(IL) ≥ n)` or `ki = 0` (probability < 1 in 2¹²⁷).

**`assumeValidity`.** `deriveHdPrivateNodeFromSeed(seed, { assumeValidity: true })` skips BIP-32's `IL < n` check on the master node — perf optimization (probability of bad master ≈ 1/2¹²⁷). The bch-bot uses it; a paranoid build should pass `false`.

## 3. BCH derivation paths

SLIP-0044 assigns **coin type `145` to Bitcoin Cash** ([satoshilabs/slips/slip-0044.md](https://github.com/satoshilabs/slips/blob/master/slip-0044.md)). Canonical BIP-44 paths:

| Path | Use |
|------|-----|
| `m/44'/145'/0'/0/i` | **Mainnet receive** (Electron Cash, Selene, Ledger, Trezor, bch-bot) |
| `m/44'/145'/0'/1/i` | **Mainnet change** |
| `m/44'/145'/a'/0/i` | Account `a`, external chain |

**Ecosystem convention (verified 2026-09-17):** bch-bot uses `m/44'/145'/0'/0/i` and `m/44'/145'/0'/1/i` ([wallet.mjs L92–98, L111–128](/home/luke/bch-bot/lib/wallet.mjs)). Selene uses `m/44'/145'/0'/0/i` for receive (`src/lib/compiler/p2pkh-utils.ts` L167–168). moth bch-wallet hardcodes `m/44'/145'/0'/0/0` as the first address — **slightly off-spec**: libauth-style addresses are `0/0`, `0/1`, `0/2`, … while moth treats `0/1` as the second account; cross-wallet restore matches only at index 0. Electron Cash uses `m/44'/145'/0'` as the root ([Ledger support docs](https://support.ledger.com/article/360009676633-zd)). Ledger accepts `m/44'/145'/0'` in Bitcoin-app BCH mode (Nano S/X whitelists coin-type 145). Trezor does not list BCH separately in [`coins-bip44-paths.md`](https://github.com/trezor/trezor-firmware/blob/main/docs/misc/coins-bip44-paths.md); ships a dedicated "Bitcoin Cash" wallet using SLIP-0044 145 + BIP-44 layout.

**Why BIP-49 is uncommon on BCH.** BIP-49 presumes SegWit (BIP-141); BCH removed SegWit at the 2018 fork — `P2SH-P2WPKH` requires a witness BCH consensus rejects. No `m/49'/145'/x'` path is meaningful on BCH. CashTokens' `p2sh32` lock is not at BIP-49 paths either. The bch-bot only derives `p2pkh`.

## 4. Change address derivation

BIP-44 prescribes a **separate change chain** (`/1`) distinct from external/receive (`/0`). The bch-bot follows this ([wallet.mjs L147–160](/home/luke/bch-bot/lib/wallet.mjs)) — every send pulls a fresh change address and increments `state.change_index`.

This matters for two reasons:

1. **Privacy.** Returning change to the same address you sent from lets the recipient (and any chain-analysis firm) link all future spends back to the original receive. Returning change to a fresh `/1/i` address breaks the link. The bch-bot's `state.json` tracks `change_index` monotonically.
2. **Signing bugs.** A wallet that confuses `/0` and `/1` at signing time can spend into the wrong chain. The bch-bot passes `change: 1` explicitly. A bug class to watch: **hardcoded `change: 0`** in any change-output path. The bch-bot does NOT have such a bug (verified) — both `newChangeAddress` and `send.mjs` use the explicit `change` parameter.

**Gap limit.** Wallets must scan `gap_limit` consecutive unused addresses on both chains to discover funds sent to old addresses. The bch-bot uses **20** for both ([wallet.mjs L163, L179](/home/luke/bch-bot/lib/wallet.mjs)) — matches BIP-44 default and moth pattern. If the gap limit is too low and the user generates >20 unused addresses in a row, funds sent to a deep address become invisible.

## 5. Address reuse privacy

Every BCH transaction is a public, immutable ledger entry. **Reusing the same address** lets any observer cluster addresses by common-input ownership (the foundational chain-analysis heuristic), link the user's identity once an address touches a KYC exchange, and track forward-flowing funds (Coinbase, Chainalysis, Elliptic, Crystal all do this commercially).

**bch-bot behavior (verified against [wallet.mjs](/home/luke/bch-bot/lib/wallet.mjs)):** `newReceivingAddress()` increments `address_index` on every call and persists to `state.json` mode 0600. Each call returns a fresh address at `m/44'/145'/0'/0/{state.address_index}`. Previous addresses remain spendable but are not auto-revealed. `newChangeAddress()` does the same on the `/1` chain. `state.json` is the source of truth for issued indices; if lost but `wallet.json` survives, the wallet recovers by re-deriving indices 0..gap_limit-1 on both chains via `deriveReceivingAddresses(20)` / `deriveChangeAddresses(20)`.

**Counter-measures beyond fresh addresses:** **CashShuffle** — collaborative transaction-shuffling added to Electron Cash; >$40M of BCH mixed ([bitcoin.com](https://news.bitcoin.com/4-bitcoin-mixers-for-the-privacy-conscious/)). **CashFusion** — successor CoinJoin-style with many participants/outputs ([cashfusion.org](https://cashfusion.org/)). **Payjoin** — sender/receiver cooperatively construct a tx including the receiver's own UTXO as input. **Stealth / reusable addresses** — specs exist ([reusable_addresses.md](https://github.com/imaginaryusername/Reusable_specs/blob/master/reusable_addresses.md)) but **no wallet ships them in production** as of 2026-09-17.

## 6. Key storage

**The seed-on-disk problem.** `wallet.json` contains `{ "mnemonic": "...", "passphrase": "..." }` written by `createWallet` ([wallet.mjs L55–56](/home/luke/bch-bot/lib/wallet.mjs)), chmod'd `0600`. **Anything that can read the file as the bot's UID owns the wallet.** This is the **default, mode-0600-only storage** moth's bch-wallet skill also uses ([moth entity](/home/luke/.openclaw/workspace/memory-bch-wiki/entities/moth-bch-wallet.md)). Moth's separate `credentials` skill uses AES-256-GCM + scrypt with `VAULT_PASS` — the bch-wallet skill does not.
**Threat model for plaintext at-rest:**

| Adversary | Read wallet.json? |
|-----------|------------------|
| Random user on host | No (mode 0600) |
| User with sudo | **Yes** — sudo reads regardless of mode |
| Rootkit / kernel compromise | **Yes** |
| Backup / snapshot system | **Yes** — often bypasses fs permissions |
| Cloud control plane (AWS, Hetzner) | **Yes** — provider can snapshot disk |

For non-trivial balances, **plaintext-at-rest is insufficient**. Counter-measures:

1. **Passphrase-encrypted wallet file** (AES-256-GCM with scrypt; PBKDF2/Argon2id acceptable). Moth's credentials skill is the model. bch-bot does **not** ship this yet.
2. **OS-level full-disk encryption** (LUKS, FileVault, BitLocker) — protects physical theft, **not** runtime root compromise.
3. **Hardware wallet** — seed never touches host filesystem. See §8.
4. **BIP-39 passphrase prompt at runtime** — gives plausible deniability if passphrase is not stored. The bch-bot currently stores the passphrase in `wallet.json` next to the mnemonic ([wallet.mjs L50](/home/luke/bch-bot/lib/wallet.mjs)) — **defeats the purpose**. Prompt at runtime, do not persist.
5. **OS keychain** (`secret-tool` / `pass`) — avoids plaintext on disk; still requires runtime read.

**Selene** uses Capacitor's `SimpleEncryption` on native (iOS Keychain, Android Keystore) — genuinely strong: PBKDF2-HMAC-SHA256 at 100 000 iterations wrapping a random AES-256-GCM data key, `.biometryCurrentSet` / `setInvalidatedByBiometricEnrollment(true)`, device-only storage mode, and an escalating PIN lockout ladder.

> **Correction (2026-10-01).** This line previously read "Desktop (Electron) uses a database-backed store with passphrase-derived keys — better than plaintext but weaker than mobile." Both halves were wrong. **Selene has no desktop/Electron build at all** (no Electron dep in `package.json`, no `src-electron/`, build docs cover only Android and iOS, and selene.cash ships only Web + `.apk`). And there is no intermediate tier: the **web** build's `encrypt()` is a passthrough that returns its input unchanged, so a wallet exported from a browser is plain JSON containing `mnemonic` and `passphrase`. Selene is therefore native-hardened **or** web-plaintext, with nothing between. Full evidence, including the format-sniff a wallet file must be checked with, in [`wallet-key-storage.md`](wallet-key-storage.md) §3.

## 7. Threat models

### 7.1 Compromised host

| Attack | Effect | Mitigation |
|--------|--------|------------|
| **Keylogger** | Captures BIP-39 passphrase; `BCH_CONFIRM=yes`; send commands | Hardware-wallet signing; air-gapped signer |
| **Clipboard hijacker** | Replaces pasted cashaddr with attacker's — single-step fund loss | Verify full address on hardware-wallet screen |
| **Memory scraper** (`/proc/<pid>/mem`) | Reads mnemonic+passphrase while unlocked | Hardware wallet; short-lived unlock with buffer zero |
| **Core dump on crash** | Core file contains plaintext mnemonic | `ulimit -c 0`; disable systemd-coredump |
| **`.bash_history` leak** | Typed mnemonic persists | `read -s` or wipe history after |
| **`/tmp` residue** | Mnemonic in debug logs | Audit all log statements |

The bch-bot's `BCH_CONFIRM=yes` env gating ([moth skill](https://dev.selene.technology/~moth/skills/bch-wallet.html)) is **not strong** — anyone reading the bot's env can spoof it. Require **out-of-band human-in-the-loop** confirmation (Telegram/Discord DM, hardware-wallet button press) for high-value sends.

### 7.2 Malicious npm dependencies

The bch-bot's runtime tree is small — **5 packages** ([package.json](file:///home/luke/bch-bot/package.json)): `@bitauth/libauth@3.0.0`, `@cashlab/cauldron@^1.0.3`, `@cashlab/common@^1.0.5`, `@electrum-cash/network@^4.3.0`, `@electrum-cash/web-socket@^4.0.3`.

**Libauth is zero-dependency** — verified at the installed `node_modules/@bitauth/libauth/package.json`: **no `"dependencies"` key**. Every line under `devDependencies` is build tooling (`ava`, `eslint`, `typescript`). A libauth supply-chain compromise must attack the package itself or its WASM blobs, not transitive deps. Blast radius is **much smaller** than Electron Cash's Python+Qt tree. The other four packages each have their own trees (worth auditing): CashLabs and electrum-cash (mainnet-pat).

**The Sept 2025 npm supply-chain attack** compromised **18 packages** including `chalk`, `debug`, `ansi-styles`, `supports-color` — >2.6B weekly downloads ([Vercel](https://vercel.com/blog/critical-npm-supply-chain-attack-response-september-8-2025), [Palo Alto](https://www.paloaltonetworks.com/blog/cloud-security/npm-supply-chain-attack/)). It was a **phishing compromise of the maintainer's npm credentials**, not a vulnerability in package source. The injected code was a **crypto-clipboard-hijacker targeting browser-based wallets** (intercepted `window.ethereum` calls). **It would not directly affect a Node.js BCH bot** (no `window.ethereum` in Node) but shows the template: phishing → maintainer creds → malicious publish → automated propagation.

The later **Shai-Hulud worm** was **install-time malware** that exfiltrated credentials and republished itself into packages the compromised maintainer had access to — propagating autonomously ([Unit 42](https://unit42.paloaltonetworks.com/monitoring-npm-supply-chain-attacks/)). One compromised maintainer account can poison the entire ecosystem graph they have publish rights to.

**Mitigations:** Pin exact versions, not semver ranges (libauth already pinned `3.0.0`; tighten CashLabs/electrum-cash from `^1.0.3` to exact pins). Audit `package-lock.json` and check it in (already done). Run `npm audit` and `socket.dev`/Snyk on every dep bump. Enable **npm 2FA for publish**; verify your dependencies' maintainers have it. For high-value bots, **vendor the deps** — copy libauth + electrum-cash source into a `vendor/` dir to remove the npm indirection.

### 7.3 Side-channel via timing

ECDSA signing implementations can leak the private key through timing variations in scalar multiplication. libauth **delegates signing to a Secp256k1 backend** ([libauth entity](/home/luke/.openclaw/workspace/memory-bch-wiki/entities/libauth.md)) — typically `@noble/secp256k1` or libauth's WASM. Both are constant-time by design, but **browser** JS cannot truly guarantee it — JIT and GC pauses introduce variance; academic cache-timing attacks have been demonstrated. **Node.js + WASM** is closer to constant-time but still subject to OS scheduling jitter. The bch-bot runs server-side in Node — lowest-risk runtime.

## 8. Counter-measures

### Passphrase encryption

Wrap `wallet.json` in AES-256-GCM with a scrypt-derived key (scrypt = moth credentials skill; PBKDF2-SHA512 600k iterations acceptable; Argon2id best). Store ciphertext, salt, nonce, and scrypt params in place of plaintext mnemonic. Prompt passphrase on each bot startup (or each send for higher security). **Implemented in bch-bot** — `createWallet({ walletPassphrase })` writes `version: 2` (scrypt + AES-256-GCM) and `loadWallet` requires the passphrase back. Covered by `scripts/test-wallet-encryption.mjs`. The gap is that it is **opt-in**: without `walletPassphrase` the file is still written as plaintext `version: 1`, so the safer path is not the default one.

### Hardware wallet integration via PSBT

BIP-174 (PSBT) hands unsigned transactions to a hardware wallet and returns them signed. BCH PSBT support is **partial**: **Ledger** (Bitcoin app in BCH mode) accepts/produces PSBTs but the Bitcoin app signs with BTC defaults; BCH requires `SIGHASH_FORKID` (0x40). **Verify on a real device before relying** — historical reports of the Bitcoin app returning signatures without forkId for non-BTC paths. **Trezor** (Model T, Safe 3, Safe 5) — BCH ships in the "Bitcoin Cash" wallet; PSBT includes forkId; programmatic signing needs `trezor-connect` or USB HID. **BitBox02** has BCH support but limited PSBT coverage for CashTokens-aware scripts.

**The BCH-specific gotcha**: any PSBT MUST carry `forkId` (0x40) on every input; token-bearing inputs MUST carry `SIGHASH_UTXOS` (0x20) — combined `0x61`. A hardware wallet that doesn't enforce this produces signatures BCH consensus rejects with `mandatory-script-verify-flag-failed`. **Test on chipnet first.**

### Air-gapped signing

For maximum security, sign on a machine that has **never touched the network**:

1. Online watch-only wallet builds the unsigned transaction, exports as PSBT (or raw hex).
2. Transfer to the air-gapped machine via **QR code**, **microSD**, or **NFC tap** — never USB that bridges networks. Coldcard's [air-gap docs](https://coldcard.com/learn/hardware-wallets/air-gapped-signing) describe the BBQr format.
3. Air-gapped machine signs (with seed or attached hardware wallet), exports signed PSBT via the same channel.
4. Online machine broadcasts.

The bch-bot does not implement air-gap mode. **No production BCH-only air-gap signer exists** as of 2026-09-17 — Coldcard and Specter DIY are BTC-only.

### Multisig

BCH supports P2SH multisig (m-of-n) and P2SH32 (CashTokens-aware). A 2-of-3 multisig (hardware + air-gapped + safe backup) eliminates single points of failure. Selene supports P2SH multisig via its `TransactionBuilderService`, though the UI flow is rough. The bch-bot does not implement multisig.

**Cost:** every signer must be reachable at broadcast time; losing one signer requires the remaining m. For 2-of-3, losing 2 signers = total loss. For 3-of-5, you can lose 2.

## 9. Real-world BCH wallet incidents

**BCH's wallet incident record is thin** — smaller market cap → smaller attacker incentive than BTC. Notable events:

- **Atomic Wallet $35M drain (June 2023)** — multi-chain; BCH holders among victims. Attributed to a compromised wallet build (supply-chain/insider compromise of Atomic's update infra), not a BIP-32/39/44 flaw. Attack vector never publicly confirmed ([Bitcoinist](https://bitcoinist.com/atomic-wallet-security-breached/amp/)).
- **Alleged $30M BCH wallet hack (Feb 2019)** — major investor posted on r/btc claiming $30M drained. Investigation inconclusive; root-cause vector never confirmed ([CoinTelegraph](https://cointelegraph.com/news/bitcoin-cash-faces-slow-death-after-alleged-30m-hack-commentator/amp)).
- **Electron Cash CashShuffle protocol bug (2020)** — flaw allowed de-anonymization; patched. Privacy-tool failure, not key-management.
- **Libauth correctness fixes** — multiple v2.x/v3.x bugfixes around signing serialization and sighash flags. None catastrophic; no known fund loss attributable.
- **`wallet.dat` legacy import CVEs** in Electron Cash — multiple historical parsing bugs. Modern wallets should not import `wallet.dat`.

**Notable absences:** no publicly known BIP-39 wordlist manipulation attack (BIP-39 checksum + first-4-letter uniqueness catches most); no publicly known BIP-32 derivation collision (~1/2¹²⁷); no publicly known cashaddr polymod checksum bypass.

**As of 2026-10-01, still true: no publicly disclosed CVE or funded-loss incident attributable to any of the five production BCH wallets researched (Cashonize, Selene, Paytaca, Electron Cash, BCHN).** Searching CVE/GHSA for these vendors returns only Bitcoin Core advisories inherited through BCHN's fork ancestry. What exists instead is *documented risk*, which is not the same as a closed vulnerability:

| Finding | Source | Status |
|---|---|---|
| Cashonize stores seed unencrypted in IndexedDB on all platforms | `cashonize-wallet/security-considerations.md` | **Declared out of scope** in `SECURITY.md`; encryption "on the roadmap" |
| Selene's web `encrypt()` is a passthrough — exported wallets are plaintext JSON | `capacitor-plugin-simple-encryption/src/web.ts:102-109` | Documented in the plugin's own header comment |
| Electron Cash wallet KDF is PBKDF2-SHA512, **1024 iterations, empty salt** | `electroncash/storage.py::get_key` | Unfixed, inherited from Electrum since 2013 |
| Electron Cash "Connect only to Preferred Servers" phishing mitigation was enabled by default, then **reverted** | `RELEASE-NOTES` 3.x → 4.0.15 | Do not rely on it |
| CashShuffle reused addresses in some outputs (de-anonymisation) | `RELEASE-NOTES` 4.0.14 | Patched |
| Electron Cash JSON-RPC security fixes | `RELEASE-NOTES` 3.1.1, 3.1.2 | Patched |
| Paytaca ships `@bitauth/libauth: 2.0.0-alpha.8` — an **alpha** crypto lib on a production wallet | `paytaca-app/package.json` | Unaddressed; watch this |

The pattern worth naming: **the BCH wallet ecosystem documents its known weaknesses rather than shipping CVEs for them.** Cashonize in particular keeps an honest `security-considerations.md` and explicitly scopes plaintext storage out of its vulnerability programme. That is better disclosure hygiene than most wallets manage, but it means a CVE search returns a misleadingly clean result — absence of CVEs here reflects absence of an audit function, not absence of risk.

## What's not solved

1. **Air-gapped BCH signing.** No production BCH-only air-gap signer with PSBT support; Coldcard and Specter DIY are BTC-only.
2. **CashTokens-aware hardware wallets.** No hardware wallet ships a P2SH32 signing path with `SIGHASH_UTXOS` (as of 2026-09). CashTokens users forced into hot wallets.
3. **Side-channel hardening of JS signing.** Browser-based wallets theoretically vulnerable to cache-timing; no production deployment addresses this.
4. **Multisig UX.** P2SH multisig exists in Selene and Electron Cash but flows are rough; users make mistakes (wrong change outputs, sighash flags) that lose funds.
5. **Passphrase-loss recovery.** Forget the BIP-39 passphrase and the funds are gone; no recovery, most wallets warn insufficiently.
6. **Encrypted-at-rest as the default.** bch-bot *can* encrypt (scrypt N=32768 + AES-256-GCM, `version: 2`) but only when a wallet passphrase is passed, so a default install is still plaintext `version: 1`. moth is the same. Selene is native-hardened but **web-plaintext** (passthrough shim). Cashonize is plaintext on every platform and says so. Paytaca is encrypted on web via a `patch-package` patch whose AES key sits in the same IndexedDB as the ciphertext. Electron Cash's default PBKDF2 is 1024 iterations with an **empty salt**. No ecosystem convergence. Evidence: [`wallet-key-storage.md`](wallet-key-storage.md).
7. **A platform shim can silently disable encryption.** Two of the four wallets researched ship a web `encrypt()` that returns its input unchanged. Any review that reads only the native crypto path will conclude they are well encrypted. Check `Capacitor.isNativePlatform()` branches and the web stub, not the mobile source.

## References

- BIP-39: <https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki>
- BIP-38: <https://github.com/bitcoin/bips/blob/master/bip-0038.mediawiki>
- BIP-32: <https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki>
- BIP-44: <https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki>
- SLIP-0044 (BCH = 145): <https://github.com/satoshilabs/slips/blob/master/slip-0044.md>
- Trezor coins-bip44-paths: <https://github.com/trezor/trezor-firmware/blob/main/docs/misc/coins-bip44-paths.md>
- Ledger BCH support: <https://support.ledger.com/article/360009676633-zd>
- Electron Cash: <https://electroncash.org/> (source: <https://github.com/Electron-Cash/Electron-Cash>)
- Cashonize: <https://github.com/cashonize/cashonize-wallet> (`security-considerations.md`, `SECURITY.md`)
- Selene encryption plugin: <https://git.xulu.tech/selene.cash/capacitor-plugin-simple-encryption>
- Paytaca: <https://github.com/paytaca/paytaca-app>
- CashFusion: <https://cashfusion.org/>
- Selene Wallet: <https://gitlab.com/selene.cash/selene-wallet>
- libauth: <https://github.com/bitauth/libauth>
- bch-bot code paths: `/home/luke/bch-bot/lib/wallet.mjs`, `/home/luke/bch-bot/lib/sign.mjs`
- Sept 2025 npm supply-chain attack: <https://www.paloaltonetworks.com/blog/cloud-security/npm-supply-chain-attack/>, <https://vercel.com/blog/critical-npm-supply-chain-attack-response-september-8-2025>
- Coldcard air-gap signing: <https://coldcard.com/learn/hardware-wallets/air-gapped-signing>
