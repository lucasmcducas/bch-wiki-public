<!-- openclaw:wiki:raw-source -->

---
pageType: source
sourceType: web
sourceUrl: https://paryonusd.com/docs/user-guides/staking
title: ParyonUSD — Staking user guide
accessedAt: 2026-07-10
description: How PUSD Stability Pool staking works end-to-end
---

# ParyonUSD — Staking User Guide (2026-07-10)

## How staking works

- You deposit PUSD into the Stability Pool
- Your stake earns yield from two sources: borrower interest payments + liquidation profits
- Yield accrues in **BCH**, claimable after each epoch
- Your stake is locked until the next epoch boundary (up to 10 days)
- Staking receipt = CashToken NFT in your wallet (shows staked amount, epoch, unclaimed payouts)

Minimum stake: 100 PUSD.

## What stakers earn

**70% of all borrower interest** flows to stakers (proportional to share). Plus liquidation gains.

### APY is backward-looking

Varies depending on:
- How much PUSD is borrowed (more = more interest)
- What interest rates borrowers pay
- How many liquidations occur
- How much PUSD is staked (larger pool = smaller share per staker)

> "The pool is self-balancing: when fewer people stake, each staker's share of yield increases, which naturally attracts more capital."

## Epochs & lock periods

- Epochs: ~10 days
- Lock period: stake is locked until next epoch boundary
- Payouts: BCH earnings available at end of each epoch, claimed separately
- Example: stake day 2 of epoch → unlocked day 10, first payout day 20
- Each epoch = separate claim transaction (no batching)

## Withdrawals

- Always full-position only (no partial)
- To reduce: withdraw fully + restake what you want
- Best to withdraw near end of epoch (captures interest already forwarded)

## Risks

### PUSD balance reduction (the main one)

When a loan is liquidated, PUSD from the pool covers the debt → your staked PUSD balance decreases.

You receive the liquidated loan's BCH collateral (at 110% MCR, the BCH is worth more than the PUSD you lost) — so in dollar terms you come out ahead.

> "In practice, this matters most during sharp BCH price drops when many loans are liquidated at once. If BCH continues to fall after you receive the collateral, the BCH compensation may end up worth less than the PUSD you lost."

### Lock period

0-10 days, depending on when in the epoch you stake.

### Variable yield

Not guaranteed. Low borrowing activity + few liquidations = low returns.

## Source URL
https://paryonusd.com/docs/user-guides/staking