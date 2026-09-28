---
pageType: entity
entityType: project
id: entity.bchn-node
aliases:
  - Bitcoin Cash Node
  - BCHN
  - bitcoincash-node
  - bitcoincashnode
sourceUrl: https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node
description: Reference full-node implementation for Bitcoin Cash (C++). Implements all BCH network upgrades including CashTokens, P2SH-32, and the May 2026 VM expansion (CHIP-2024-12 P2S, bounded loops, functions, bitwise).
claims:
  - id: claim.bchn.language
    text: Bitcoin Cash Node is implemented primarily in C++ (~70% of codebase), with Python ~17% (functional tests), C ~7%, and the rest CMake/Shell.
    status: supported
    confidence: 0.95
    evidence:
      - kind: api-data
        sourceId: source.bchn-gitlab
        path: sources/bchn-gitlab.md
        weight: 1.0
  - id: claim.bchn.maintainer
    text: BCHN is maintained by the Bitcoin Cash Node team, a miner-friendly professional node project that forked from Bitcoin ABC.
    status: supported
    confidence: 0.9
    evidence:
      - kind: direct-read
        sourceId: source.bchn-gitlab
        path: sources/bchn-gitlab.md
        weight: 0.9
  - id: claim.bchn.latest-release
    text: Latest release is v29.1.0 (2026-07-28). v29.0.0 (2026-01-09) activated the May 15, 2026 network upgrade.
    status: supported
    confidence: 0.95
    evidence:
      - kind: api-data
        sourceId: source.bchn-gitlab
        path: sources/bchn-gitlab.md
        weight: 1.0
  - id: claim.bchn.cashtokens-rpc-lowlevel
    text: CashTokens support in BCHN is explicitly low-level as of v29.1.0. The wallet does NOT track user tokens. Tokens are only manageable via RPC; they persist only in the UTXO DB and block DB. There is no GUI token management.
    status: supported
    confidence: 0.95
    evidence:
      - kind: direct-read
        sourceId: source.bchn-v29-release-notes
        path: sources/bchn-release-notes.md
        weight: 1.0
  - id: claim.bchn.cashtokens-rpc-methods
    text: CashTokens-aware RPC methods added across v25–v28: gettxout (tokenData field), decoderawtransaction, getrawtransaction (tokenData + scriptPubKey at verbosity=2 since v28), decodepsbt (input utxo tokenData), listunspent (filter options), signrawtransactionwithkey and signrawtransactionwithwallet (tokenData on prevouts), createrawtransaction, getdsproofscore.
    status: supported
    confidence: 0.9
    evidence:
      - kind: direct-read
        sourceId: source.bchn-v25-release-notes
        path: sources/bchn-release-notes.md
        weight: 0.95
  - id: claim.bchn.bytecode-patterns
    text: v29.0.0 added a 'patterns' flag to getblock and getrawtransaction RPCs (and new REST endpoints /block/withpatterns/<HASH>, /tx/withpatterns/<HASH>) that emits a byteCodePattern JSON object describing the script's pattern and fingerprint — useful for smart-contract categorization.
    status: supported
    confidence: 0.9
    evidence:
      - kind: direct-read
        sourceId: source.bchn-v29-release-notes
        path: sources/bchn-release-notes.md
        weight: 1.0
  - id: claim.bchn.sighash-utxo
    text: BCHN extended sighash types with SIGHASH_UTXO (0x20) for the CashTokens spec; not covered by DSProofs as of v29.1.0.
    status: supported
    confidence: 0.85
    evidence:
      - kind: direct-read
        sourceId: source.bchn-v29-release-notes
        path: sources/bchn-release-notes.md
        weight: 0.9
  - id: claim.bchn.may-2026-upgrade
    text: v29.0.0 implements the May 15, 2026 network upgrade: CHIP-2024-12 P2S (Pay to Script), CHIP-2021-05-loops (bounded loops), CHIP-2025-05 functions, CHIP-2025-05 bitwise operations re-enable.
    status: supported
    confidence: 0.95
    evidence:
      - kind: direct-read
        sourceId: source.bchn-v29-release-notes
        path: sources/bchn-release-notes.md
        weight: 1.0
---

# Bitcoin Cash Node (BCHN)

**Type**: Reference full-node implementation
**GitLab**: https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node
**Site**: https://bitcoincashnode.org
**Default branch**: `master`
**License**: Project source is MIT-style but the GitLab project itself shows `license: None` (check `COPYING` / individual file headers before commercial reuse)
**Latest release**: v29.1.0 (2026-07-28). Activity last confirmed 2026-09-14.

A professional, miner-friendly full-node implementation for Bitcoin Cash. Forked from Bitcoin ABC in 2020-02 after the ABC/Node split over the IFP (infrastructure funding plan). Implements every BCH consensus upgrade to date, including CashTokens (May 2023), P2SH-32, BigInt + VM Limits (May 2025), and the May 2026 upgrade (P2S / loops / functions / bitwise).

## Language composition

Per GitLab Languages API (`/api/v4/projects/.../languages`):

| Language | % |
|---|---|
| C++ | 70.49% |
| Python | 17.24% (functional tests + a few utilities) |
| C | 7.02% (crypto primitives, secp256k1, libsecp256k1 bindings) |
| CMake | 1.38% |
| Shell | 1.17% |

**Why C++ (and is it preferable)?**

Yes, for a node implementation, C++ is the right call — and the historical norm:

| Reason | Why it matters |
|---|---|
| **Performance** | A full node validates every transaction, every script, every signature at chain tip. C++ gives sub-millisecond script execution and lets you keep the UTXO set hot in RAM. Rust/Go are credible alternatives today; Python/JS would not be. |
| **Existing ecosystem** | BCHN forked from Bitcoin Core which was C++ since 2009. Reusing secp256k1, leveldb, libevent, and a decade of optimization work is the lowest-cost path. |
| **Hardened consensus code** | The Bitcoin-derived full-node ecosystem is reviewed by the same set of security researchers who read C++. Switching languages forces a re-review of consensus-critical code. |
| **FFI to crypto primitives** | secp256k1, libcrypto, leveldb are all C/C++. Cheap to link. |
| **Build maturity** | CMake + autotools work on every platform miners actually deploy (Linux ARM servers, macOS dev, Windows server). |

**Downsides of C++ here:**
- Memory-safety bugs are an attack surface. Buffer overflows in script deserialization have been a recurring class of CVEs across Bitcoin-derived code.
- Build times and dependency hell (BCHN pulls in BerkeleyDB, Boost, libevent, libsecp256k1, ZeroMQ, etc.).
- Onboarding new contributors is harder than for Go/Rust.

**For our LLM-training purposes, C++ is exactly what we want in the corpus** — it forces the model to learn real Script opcodes, real sighash flag handling, real UTXO set manipulation, and real P2SH template construction, not hand-wavy high-level abstractions.

## CashTokens support on mainnet — RPC reality (yes, low-level)

This is the honest state of CashTokens support in BCHN as of v29.1.0 (verified from the v28.0.0 and v29.1.0 release notes' explicit "Limitations" section — the wording is nearly identical, which is itself a tell):

> "CashToken support is low-level at this stage. The wallet application does not yet keep track of the user's tokens. Tokens are only manageable via RPC commands currently. They only persist through the UTXO database and block database at this point. There are existing RPC commands to list and filter for tokens in the UTXO set. RPC raw transaction handling commands have been extended to allow creation (and sending) of token transactions. Interested users are advised to consult the functional test in `test/functional/bchn-rpc-tokens.py` for examples on token transaction construction and listing. **Future releases will aim to extend the RPC API with more convenient ways to create and spend tokens, as well as upgrading the wallet storage and indexing subsystems to persistently store data about tokens of interest to the user. Later we expect to add GUI wallet management of Cash Tokens.**"

What this means for builders today:

1. **No `listtokens`, no `gettokenbalance`, no `sendtoken`.** The wallet subsystem does not maintain a token balance. If you want "balance of PUSD I own", you have to call `listunspent` (filter by `tokenCategory`) and sum the amounts yourself.
2. **No persistent token index.** Tokens are in the UTXO DB only. Rescan from genesis is needed if your wallet loses sync.
3. **No GUI token management.** `bitcoin-qt` shows sat balances, not token balances.
4. **Construction is raw.** You build token transactions with `createrawtransaction` (with the `data` array for token mint/burn/transfer outputs), fund with `fundrawtransaction` (which is token-aware for inputs), and sign with `signrawtransactionwithwallet` passing `tokenData` on each prevout.
5. **`scantxoutset` is your friend.** It's the canonical "which UTXOs are owned by address X and carry token Y" tool. BCHN added token filter options to it for CashTokens.

### Which RPC methods actually expose CashTokens?

(Confirmed from v25.0.0 and v28.0.0 release notes.)

| RPC | CashTokens-related change |
|---|---|
| `gettxout` | Returns `tokenData` JSON object if the output contains token data (v25); `byteCodePatterns` (v29.0.0); new 4th arg `patterns` (v29.1.0) |
| `getrawtransaction` (verbosity 2) | Returns `tokenData` + `scriptPubKey` on each output (v28); supports `patterns=true` (v29.0.0) |
| `decoderawtransaction` | Returns `tokenData` on outputs |
| `decodepsbt` | `input.utxo` extended with optional `tokenData` |
| `listunspent` | Two new boolean options controlling listed results (token filtering) |
| `signrawtransactionwithkey` / `signrawtransactionwithwallet` | Accept `tokenData` on prevout objects |
| `createrawtransaction` | Accepts `data` array for token mint/burn outputs |
| `scantxoutset` | Token filter options |
| `getblock` (verbosity ≥2) | `patterns=true` adds `byteCodePattern` to scripts (v29.0.0) |
| `getblockchaininfo` | New `upgrade_status` field (v29.0.0) |
| `getindexinfo` | New RPC; returns indexer status (v28.0.0) |

### What's NOT in BCHN (and why this matters for our LLM)

- No `gettokeninfo`, no `gettokentransactions`, no `gettokenholders` — full token indexers live outside the node (e.g. Cauldron's indexer, PSF's token endpoints, Fulcrum, Electrum Cash). Building a wallet means **choosing an indexer**.
- No built-in CashScript-aware endpoints. The CashScript SDK does its own UTXO fetching via Fulcrum/Electrum-Cash and constructs transactions that it hands to BCHN for `sendrawtransaction`.
- No built-in oracle integration. AnyHedge oracles are external services.
- No `getblocktemplate` token-aware outputs beyond what miners need.

## Script fingerprinting (v29.0.0) — directly relevant to "smart contract context"

`byteCodePattern` is a new RPC field (and REST endpoint field) that summarizes a script's "shape" — e.g. `p2pkh`, `p2sh20`, `p2sh32`, `multisig`, `op_return_data`, `covenant_*` — with a fingerprint for grouping. This is gold for:

- Block explorers (categorize scripts without re-disassembling).
- Wallet UIs (render "this is a P2SH contract" badges).
- **Our LLM training corpus** — labeled samples of common BCH script patterns.

Reference discussion: https://bitcoincashresearch.org/t/smart-contract-fingerprinting-a-method-for-pattern-recognition-and-analysis-in-bitcoin-cash/1441

## Recent consensus upgrades implemented

| Upgrade | Version | What it added |
|---|---|---|
| CashTokens (CHIP-2022-02) | v25.0.0 (May 2023) | Native FT/NFT tokens, covenants, SIGHASH_UTXO |
| P2SH-32 (CHIP-2022-05) | v25.0.0 | 32-byte hash P2SH for stronger smart-contract collision resistance |
| VM Limits + BigInt | v28.0.0 (May 2025) | Expanded Script VM limits, arbitrary precision ints up to 80,000 bits |
| P2S + Loops + Functions + Bitwise | v29.0.0 (May 2026) | CHIP-2024-12 Pay-to-Script, bounded loops, function ops, bitwise ops re-enabled |

The May 2026 upgrade is the biggest since CashTokens — bounded loops + functions in particular open up entire new classes of on-chain covenants (looping, recursion-free but composable). Any BCH LLM fine-tune MUST train on these post-2026 opcodes; a corpus that stops at v25 will be ~2 years stale by release.

## Releases (verified from GitLab API)

| Tag | Released | Headline |
|---|---|---|
| v29.1.0 | 2026-07-28 | RPC improvements (rest/spenttxouts, byteCodePatterns on gettxout), perf optimizations |
| v29.0.0 | 2026-01-09 | May 15, 2026 network upgrade (P2S, loops, functions, bitwise) |
| v28.0.1 | 2024-12-29 | Bug fixes |
| v28.0.0 | 2024-11-30 | May 15, 2025 upgrade (VM limits, BigInt) |
| v27.1.0 | 2024-07-10 | Maintenance |

## Why this matters for our BCH smart-contract LLM

1. **C++ in the corpus is mandatory.** Script semantics are encoded in C++ in `src/script/`. Without reading that code, the model cannot learn what `OP_CHECKSIG` actually does on BCH (vs BTC) — sighash prefix `0x41`, ANYONECANPAY/ANYPREVOUT semantics, the way `SIGHASH_UTXO` interacts with token prevouts.
2. **The "no high-level RPC for tokens" gap is the real product opportunity.** A wallet that talks to BCHN's *raw* CashTokens RPC and turns it into a normal "balance of PUSD = X, send Y to Z" experience is non-trivial — that's where Moth's BCH wallet skill and the planned mobile app have to live. The LLM should be aware of this gap and know how to construct the raw RPC calls.
3. **`bchn-rpc-tokens.py` is a hidden goldmine** for SFT pairs. It's a functional test that demonstrates token construction end-to-end. Every example in there is a verified-correct training example.

## Sources

- GitLab repo metadata + languages: `sources/bchn-gitlab.md`
- v29.1.0 + v29.0.0 + v28.0.0 + v25.0.0 release notes: `sources/bchn-release-notes.md`
- Functional test for token construction: `test/functional/bchn-rpc-tokens.py` (in the repo)

## Related

- [CashTokens](concepts/cash-tokens.md)
- [BCH Knowledge Base](entities/bch-knowledge-base.md)
- [PSF LLM Wiki](entities/psf-llm-wiki.md)
- [Cauldron DEX](entities/cauldron-dex.md) — uses BCHN-style raw token RPCs for its indexer

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
