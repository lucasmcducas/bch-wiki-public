---
pageType: entity
entityType: protocol
id: entity.paryonusd
aliases:
  - PUSD
  - Paryon USD
  - Paryon
sourceUrl: https://paryonusd.com
description: Decentralized over-collateralized stablecoin on Bitcoin Cash (CashTokens), modeled on Liquity V2
claims:
  - id: claim.pusd.tokenid
    text: PUSD is a fungible CashToken on BCH mainnet, token ID 2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544, 2 decimals.
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-launch
        path: sources/paryonusd-launch.md
        weight: 1.0
  - id: claim.pusd.model
    text: PUSD is a Liquity V2 fork: over-collateralized loans with BCH, Stability Pool, dynamic market-driven interest rates, direct redemption.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-landing
        path: sources/paryonusd-landing.md
        weight: 0.9
  - id: claim.pusd.yield-split
    text: 70% of borrower interest goes to Stability Pool stakers (paid in BCH); the rest goes to protocol/treasury. Liquidations also distribute collateral (BCH) to stakers at the 110% collateral ratio.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-staking
        path: sources/paryonusd-staking.md
        weight: 1.0
  - id: claim.pusd.epoch
    text: Stability Pool uses epochs (~10 days). Stakes are locked until next epoch boundary. Withdrawals are full-position only.
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-staking
        path: sources/paryonusd-staking.md
        weight: 1.0
  - id: claim.pusd.minimums
    text: Minimum loan 100 PUSD, minimum stake 100 PUSD, minimum collateral ratio ultimately 110% (launch started at 180% / max LTV 55.6%).
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-launch
        path: sources/paryonusd-launch.md
        weight: 1.0
  - id: claim.pusd.launch
    text: Launched live April 30, 2026. Redemptions disabled at launch, enabled in follow-up step.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.paryonusd-launch
        path: sources/paryonusd-launch.md
        weight: 1.0
---

# ParyonUSD (PUSD)

**Type**: Decentralized stablecoin protocol on Bitcoin Cash
**Ticker**: PUSD
**Website**: https://paryonusd.com
**Token ID**: `2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544`
**Decimals**: 2
**Launched**: April 30, 2026
**Underlying model**: Liquity V2 fork on CashTokens

---

## What it is

ParyonUSD is the first major Liquity-style stablecoin on Bitcoin Cash. Users lock BCH as collateral and mint PUSD. PUSD is a fungible CashToken, native to BCH at consensus level.

## Core mechanics

| Action | What happens |
|--------|--------------|
| **Borrow** | Deposit BCH collateral → mint PUSD. Min collateral ratio 110% (was 180% at launch). |
| **Stake** | Deposit PUSD into Stability Pool → earn 70% of borrower interest paid in BCH + liquidation gains. Locked until next epoch. |
| **Redeem** | Trade PUSD for BCH at face value (maintains $1 peg). Disabled at launch; to be enabled in follow-up step. |
| **Liquidate** | Troves below 110% get liquidated; PUSD from pool covers the debt, pool receives BCH collateral. |

## Yield for PUSD holders

- **70% of borrower interest** flows to Stability Pool stakers (paid in BCH)
- **Liquidation gains** = the gap between BCH collateral received (worth >$110 per $100 of debt at 110% MCR) and PUSD used
- APY is backward-looking, varies with: borrow volume, average borrow rate, number of liquidations, pool size
- Yield is **real** (paid in BCH by borrowers), not inflationary

## Lock-up & epochs

- Epochs are ~10 days
- New stakes are locked until the next epoch boundary (0-10 day wait)
- First BCH payout is claimable at the end of the following epoch (10-20 days from staking)
- Each epoch = one payout claim transaction
- Full-position withdrawals only; partial reductions require withdraw+restake

## Minimums

- Min loan: 100 PUSD
- Min stake: 100 PUSD
- Min collateral ratio: 110% (180% at launch)

## Source code & security

- Contracts: github.com/ParyonUSD/contracts
- Two external audits, plus follow-up AI-driven soundness reviews
- Post-audit hardening changes published
- Standalone deployment verification tool: github.com/ParyonUSD/verify_contract_deployment

## Strong points
- CashTokens native → fees are negligible (~$0.001/tx)
- Censorship resistant, no blacklisting
- Direct redemption mechanic → strong peg guarantees
- Multiple DEX integrations (Cauldron, etc.)

## Weak points / frictions
- Epoch lock-up = poor UX for users who want to move money freely
- Each epoch = separate claim transaction → annoying if you don't claim often
- "Stake is locked" framing → feels restrictive vs. "earn interest on savings"
- Minimum 100 PUSD = filters out small holders
- No mobile-first UX today (web app only)
- Yield is variable → no guaranteed rate, harder to market
- Stability Pool risk: PUSD balance shrinks during BCH crashes (even if $ value goes up)

## Related pages
- [Cauldron](entities/cauldron-dex.md) — DEX where PUSD/BCH trades
- [CashTokens](concepts/cash-tokens.md)
- [Cash Stack](concepts/cash-stack.md)
- [Source: PUSD landing](sources/paryonusd-landing.md)
- [Source: PUSD launch blog](sources/paryonusd-launch.md)
- [Source: PUSD staking docs](sources/paryonusd-staking.md)