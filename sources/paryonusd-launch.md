<!-- openclaw:wiki:raw-source -->

---
pageType: source
sourceType: web
sourceUrl: https://paryonusd.com/blog/launch
title: ParyonUSD — Launch Day blog post
accessedAt: 2026-07-10
description: Official launch announcement for ParyonUSD on BCH mainnet
published: 2026-04-30
---

# ParyonUSD — Launch Day (2026-04-30)

ParyonUSD is live on Bitcoin Cash mainnet. ~1 year after announcement at BCH Bliss 2025.

## Token
- Token ID: `2469acc5afa4b10cb5b5c04afb89c3a3ffd61c5da9c01e26d00951cae2a02544`
- 2 decimals (smallest unit = 0.01 PUSD)

## Stats page
https://stats.paryonusd.com/ — TVL, PUSD supply, stability pool, active loans, redemptions

## Protocol Minimums (hard-coded in contracts)
- Minimum loan: 100 PUSD
- Minimum stake: 100 PUSD

## Launch Window Restrictions
- **Max LTV: 55.6%** (min 180% collateral ratio). New loans opened at ≥180% CR. Safety buffer while stability pool grows.
- **Redemptions disabled** at launch. Enabled in follow-up step.

## Launch Window Dynamics
> "In the meantime, the peg is supported by the dynamic interest rate and by the well-funded stability pool ready to absorb liquidations. PUSD can be traded against BCH on CashTokens DEXes as usual."

## Security
- Two external audits
- Plus ongoing AI-driven soundness reviews
- Post-audit hardening changes: https://github.com/ParyonUSD/contracts/blob/main/post-audit-changes.md
- Standalone deployment verification tool: https://github.com/ParyonUSD/verify_contract_deployment

## Source URL
https://paryonusd.com/blog/launch