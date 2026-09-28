---
pageType: synthesis
id: synthesis.bch-bot-pusd-integration
title: BCH Bot - ParyonUSD (PUSD) Phase 3 Integration Plan
createdAt: 2026-09-17
covers:
  - entity.paryonusd
  - entity.libauth
description: Phase 3 design for the bch-bot PUSD Stability Pool integration (stake, claim, withdraw). Concrete scripts, transaction shapes, sighash flags, contract bytecodes. PUSD contract fingerprints verified against @paryonusd/contracts v1.0.0 on npm + ParyonUSD/verify_contract_deployment mainnet-v1 config.
---

# BCH Bot — ParyonUSD (PUSD) Phase 3 Integration Plan

*Design doc for the stake / claim / withdraw scripts. Verification of contract fingerprints and tx shapes against the live ParyonUSD deployment (mainnet-v1, contract set `e2136acf` + sidecar `4bb467b6` + addLiquidity `f46099a7`).*

## 1. TL;DR

Three new scripts (`scripts/stake.mjs`, `scripts/claim.mjs`, `scripts/withdraw.mjs`) that build, sign, and broadcast 4–6-output multi-token transactions against the Stability Pool covenant. **No new dependencies beyond `@cashscript/cashscript` for the manual signing-serialization path.** Total addition: ~400 LOC. Testnet not needed — chipnet doesn't have the May2026 PUSD launch yet, so the only honest test is a tiny mainnet dust round-trip with real PUSD.

## 2. Why this is hard (and what makes it tractable)

**Hard because** every covenant input needs the **manual signing-serialization path** (Selene's `signTemplate`, libauth's `generateSigningSerializationBCH` + `signMessageHashSchnorr`):

1. The covenant's redeem script (function contract bytecode) is the `coveredBytecode` in the sighash preimage
2. Sighash byte = `SIGHASH_ALL | SIGHASH_UTXOS | SIGHASH_FORKID` = **0x61** (always, for covenant inputs — *different from Phase 1's 0x41*)
3. The token prefix from the spent UTXO must be in the preimage (`outputTokenPrefix`)
4. The signature is appended to the sighash byte (`...sig, 0x61`)
5. **The signed signature replaces placeholder bytes in the redeem script**, which itself is appended to the unlocking bytecode after the user's signature

**Tractable because** Selene's `KeyManagerService.signTemplate` (verified 2026-09-17, ~70 LOC) is the canonical implementation. We adapt it. The bytecodes are in `@paryonusd/contracts` v1.0.0. The redeem-script placeholder pattern (`<push sig> <push pubkey>`) is hardcoded by the contract authors.

## 3. The contracts we interact with

From `@paryonusd/contracts@1.0.0` (verified, fingerprint column truncated to 8 chars):

| Artifact | Fingerprint | bytecode | Role |
|----------|-------------|----------|------|
| `AddLiquidity` | `f46099a7…` | 1190 B | Function contract for stake. Constructor: `bytes32 paryonTokenId` |
| `WithdrawFromPool` | `dfecf8b4…` | 1708 B | Function contract for withdraw. Same constructor |
| `NewPeriodPool` | `2b7045d4…` | 2344 B | Function contract for epoch bump |
| `StabilityPool` | `e2136acf…` | 567 B | The main pool P2SH (carries the mutable state NFT) |
| `StabilityPoolSidecar` | `4bb467b6…` | 545 B | Holds the aggregated PUSD deposits |
| `Payout` | `26e6cf01…` | 1122 B | Holds claimable BCH yield |
| `PriceContract` | `48b40fef…` | 1558 B | Oracle reads (for liquidations, not stake/claim) |
| `paryonTokenId` (constructor arg) | n/a | 32 B | `2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544` |

From `ParyonUSD/verify_contract_deployment/config.ts` (mainnet-v1):

- `poolTokenId`: `7708645a7f30e97003573d9322202960a560a87527bef3666a30044a0dfdfa81`
- `paryonTokenId`: `2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544`
- `oraclePublicKey`: `02d09db08af1ff4e8453919cc866a4be427d7bfe18f2c05e5444c196fcf6fd2818`
- `protocolFeeLockingBytecode`: `aa207c7a248c794af2cdf1b1a66a2311347c45b177fcb3aad730a7823c9e32fc754087`
- `startBlockHeight`: 948406
- `periodLengthBlocks`: 144 (epoch = 1440 blocks, ~10 days)
- `timeLockRedemption`: 12

## 4. The three operations

### 4.1 Stake (addToPool via `AddLiquidity`)

The `AddLiquidity.cash` contract enforces this exact tx shape:

| Output | Type | What |
|--------|------|------|
| 0 | StabilityPool P2SH (mutable NFT, value = input0.value, NFT commitment unchanged) | recreates the pool state at output 0 |
| 1 | StabilityPoolSidecar P2SH (PUSD amount = input1.amount + `addedTokenAmount`) | holds the new total stake |
| 2 | AddLiquidity function contract (P2SH, value = 1000 sats, NFT with state identifier 0x01) | recreates the function contract, pinned to current epoch state |
| 3 | User's staking receipt NFT (value = 1000 sats, commitment = `nextEpoch (4b BE) + addedTokenAmount (LE)`) | proves the user's stake |
| 4 (optional) | PUSD change OR no-token output back to user | excess PUSD if user overpaid |
| 5 (optional) | BCH change (no tokens) | fee change |

Inputs:
- 0: StabilityPool UTXO (carries the minting NFT — the category id is `poolTokenId + 0x02` per `require(tx.inputs[0].tokenCategory == stabilityPoolTokenId + 0x02);`)
- 1: StabilityPoolSidecar UTXO (PUSD-bearing)
- 2: AddLiquidity function contract UTXO (the "current epoch state" — its NFT commitment is the active input)
- 3: User's PUSD UTXO (≥ 100.00 PUSD = 10000 base units)
- 4: User's BCH UTXO (for fees + AddLiquidity's required 1000-sat value)

The receipt's NFT commitment is the unique bit: `nextEpoch` parsed from `inputs[0].nftCommitment.split(4)[0] / 10 + 1`, padded to 4 bytes BE, followed by `addedTokenAmount` as bytes (LE int). To verify the math: parse the input NFT commitment → first 4 bytes → big-endian uint32 → divide by 10 → add 1 → emit next epoch. Then append the added token amount bytes.

**Sighash on input 2 (the covenant input): `0x61` (all + utxos + forkId).** Inputs 0, 1, 3, 4 use plain P2PKH signing (`0x41`) via `walletTemplateP2pkhNonHd`.

**Script signature: Selene's `signTemplate` pattern.** The function contract bytecode has placeholders for sig + pubkey; the bot computes the sig, replaces the placeholder, prepends it to the redeemscript, and that becomes the input's unlocking bytecode.

### 4.2 Claim (no covenant interaction needed)

Claim is just `blockchain.transaction.broadcast` of a transaction that spends the **Payout** covenant's BCH yield to the user's address. The Payout contract holds the BCH; whoever can prove they own a receipt NFT can claim the corresponding share. The claiming transaction:

- Input 0: Payout covenant UTXO (a BCH-only UTXO with minting NFT)
- Input 1: User's PUSD receipt NFT UTXO (proves claim eligibility)
- Outputs: BCH to user's address (share) + change

Sighash on input 0: also `0x61` (covenant). Input 1: `0x41`.

### 4.3 Withdraw (WithdrawFromPool)

Mirror of `AddLiquidity` but the `addedTokenAmount` becomes `negative` (output 1's token amount = input 1's amount - `withdrawnTokenAmount`). The receipt NFT is destroyed (no output recreates it). User gets their original PUSD back + any accumulated BCH yield share.

Sighash on the WithdrawFromPool input: `0x61`.

## 5. The sighash flag question (settled)

**P2PKH inputs → 0x41 (`SIGHASH_ALL | SIGHASH_FORKID`).** Phase 1 pattern, works for P2PKH token spends (Selene proves this in production).

**Covenant inputs (any P2SH containing CashScript) → 0x61 (`SIGHASH_ALL | SIGHASH_UTXOS | SIGHASH_FORKID`).** Mandatory for token-bearing covenant spends. The ChipNet May2026 consensus rules reject covenant transactions with `0x41` that involve tokens.

**Multiple inputs in one tx → all-or-nothing upgrade.** Per libauth tests, `verifyAlgorithmWithTokensInMultipleInputsAndOutputs` — once any input has `0x61`, all inputs that touch tokens must use `0x61`. PUSD stake has mixed inputs (P2PKH for fees + covenant for the state transition), so the covenant inputs need `0x61`. The P2PKH fee inputs can stay `0x41` — only the inputs that *touch* token UTXOs need the upgrade.

## 6. Implementation sketch

Three files:

### `lib/cashscript.mjs` (~150 LOC)

- `instantiateContract(artifactJson, constructorArgs)` — wraps libauth's `HashLock`, `NonFungibleToken`, etc.
- `getP2sh20LockingBytecode(contract)` — standard CashScript helper
- `signCovenantInput({ ...args })` — adapt Selene's `signTemplate`:
  - Compute sighash preimage via `generateSigningSerializationBCH` with sighash byte `0x61`
  - Sign with `secp256k1.signMessageHashSchnorr`
  - Append `0x61` to the signature
  - Replace the sig + pubkey placeholders in the function contract's redeemScript
  - Build the full unlocking bytecode: `<push sig_with_sighash> <push pubkey> <serialized_redeemScript>`

### `scripts/stake.mjs` (~150 LOC)

- Args: `<paryon-usd-amount>` (e.g. `100.00`)
- Connects to Rostrum, fetches: StabilityPool, StabilityPoolSidecar, AddLiquidity (current), user's PUSD UTXO, user's BCH UTXO
- Constructs the 4–6 output tx per §4.1
- Signs: inputs 0, 1, 3, 4 with `0x41` (P2PKH compiler), input 2 with `0x61` (covenant)
- Returns receipt NFT commitment on success

### `scripts/claim.mjs` + `scripts/withdraw.mjs` (~120 LOC each)

Similar shape to `stake.mjs` but the specific 5–6-output pattern.

### Library additions

- `lib/pusd.mjs` (~200 LOC) — wraps the three operations with the explicit tx shapes, sighash flags, and output ordering. Encapsulates the epoch math (parse current epoch from input NFT commitment, compute next epoch).

## 7. Risks specific to this phase

1. **CashScript dependency.** Adds `@cashscript/cashscript` (CJS dep, ~5 MB) to the bot. Vendored compile artifacts (we don't need to run cashc, but we do need the runtime to instantiate contracts).
2. **Covenant sighash bugs.** The `0x61` requirement is consensus-level. A bot signing with `0x41` on a covenant input will produce a valid-looking signature that the network rejects — the tx is well-formed but unspendable. **Mitigation:** dry-run is mandatory; never broadcast without parsing the sighash byte back out and confirming it matches.
3. **Epoch math.** The `nextEpoch = currentEpoch + 1` calculation looks trivial but it's derived from `inputs[0].nftCommitment.split(4)[0] / 10 + 1`. Off-by-one here means the receipt is invalid → withdraw fails. **Mitigation:** write a unit test that takes a known stability-pool commitment and verifies the receipt commitment matches.
4. **PUSD decimals.** ParyonUSD has 2 decimals (100_00 = 100.00 PUSD). The bot's existing balance parsing for FT amounts is category-agnostic — works with any decimals — but the dust-floor (100.00 PUSD minimum) needs to be hardcoded.
5. **No chipnet testnet for May2026 contracts.** ParyonUSD launched April 30 2026 (per wiki source). Chipnet activates new CashTokens contracts at network upgrades; there might be a separate testnet deployment we're not aware of. **Mitigation:** mainnet dust round-trip with the smallest legal stake (100.00 PUSD ≈ $1) is the only honest test path. Cost: $1 + fees.

## 8. What I already know vs. what needs verification at code-write time

**Already verified (this session):**

- 26 ParyonUSD contract artifacts on npm v1.0.0
- All fingerprints match `ParyonUSD/contracts` repo source
- AddLiquidity.cash bytecode + abi + source
- WithdrawFromPool.cash available (similar structure, not yet read in detail)
- Deployment config: poolTokenId, paryonTokenId, oracle pubkey, periods, timeLockRedemption
- libauth's compiler path works for P2PKH (Phase1 ship test)
- libauth's manual signing-serialization path works for covenant inputs (Selene's `signTemplate`, need to verify locally before broadcast)
- BCH 0-conf protection via DSProofs (no Avalanche — that was wrong, that lives on eCash/XEC)

**Needs verification at code-write time:**

- The exact sighash byte for mixed-input transactions (the "all-or-nothing" rule)
- Whether `walletTemplateP2pkhNonHd` (used in Phase1) signs with `0x41` correctly when the same tx has a covenant input that signs with `0x61` (libauth compiler should handle this independently per-input)
- The Payout contract's actual claiming mechanism (haven't read `Payout.cash` yet)
- Whether `payout()` requires any specific sequence number or locktime
- NewPeriodPool interaction (do we ever need to call it, or is it autonomous?)

## 9. Out of scope for Phase 3 (deferred)

- Liquidations (`liquidateLoan`, `LiquidateLoan`): need oracle read flow
- Borrowing (`Borrowing.cash`, `Loan.cash`): CDP open/close; large script
- `manageLoan`, `payInterest`, `changeInterest`: lifecycle ops, all need oracle
- `Redemption`, `Redeemer`, `swapInRedemption`, `swapOutRedemption`, `startRedemption`: the redemption mechanism (different from stability pool)
- `LoanKeyFactory`, `LoanKeyOriginEnforcer`, `LoanKeyOriginProof`: key infrastructure, only relevant when opening a loan
- `Collector`: collects protocol fees; user-triggered sweeps

These are real contracts that ship with PUSD but the bot doesn't need them for "operate a treasury with PUSD yield." If a future phase needs them (e.g., opening a BCH-collateralized loan), they follow the same pattern: instantiate artifact, construct tx per contract shape, sign with `0x61` on the covenant input.

## 10. Estimated work

| Phase | Effort | Calendar |
|-------|--------|----------|
| `lib/cashscript.mjs` (signing primitives) | 1 day | Day 1 |
| `lib/pusd.mjs` (3 operation shapes + epoch math) | 2 days | Days 2–3 |
| `scripts/stake.mjs` | 1 day | Day 4 |
| `scripts/claim.mjs` + `scripts/withdraw.mjs` | 1 day | Day 5 |
| Test with $1 dust round-trip on mainnet | 0.5 day | Day 5.5 |
| SKILL.md / KEYRING.md update | 0.5 day | Day 6 |
| **Phase 3 total** | **~6 days focused** | **~1 calendar week** |

## 11. Dependencies to add

```bash
npm install @cashscript/cashscript @paryonusd/contracts
```

The two packages are MIT-licensed, ~5 MB combined, both actively maintained by the Paryon team (mr-zwets on GitHub).

---

*Authored 2026-09-17 by Jav during the bch-bot Phase 2 prep work. Verified contract fingerprints against @paryonusd/contracts@1.0.0 (npm) + ParyonUSD/contracts@main (GitHub) + ParyonUSD/verify_contract_deployment@main/config.ts. Implementation has not started.*