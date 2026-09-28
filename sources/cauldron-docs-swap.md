<!-- openclaw:wiki:raw-source -->

---
pageType: source
sourceType: web
sourceUrl: https://docs.riftenlabs.com/cauldron/swap
title: Cauldron DEX — Trade/Swap mechanics
accessedAt: 2026-07-10
description: Technical explanation of how swaps work on Cauldron
---

# Cauldron — Trade/Swap Mechanics (2026-07-10)

## How trades happen

Trades interact with public Cauldron Liquidity Pools on BCH. A trade buys/sells tokens from liquidity contracts deployed by Cauldron users.

**Fee: 0.3% to liquidity providers on every trade.**

## Simple trade structure

```
BCH → Token:
  CauldronInput  → TradeTx → CauldronOutput
  UserBCHInput   → TradeTx → UserTokenOutput
                  TradeTx → UserBCHChangeOutput
```

The tx consumes a Cauldron UTXO as input and re-creates it at the same output index.

## Additional liquidity

Multiple Cauldron contracts can be used in one transaction:

```
  CauldronInput1 → TradeTx → CauldronOutput1
  CauldronInput2 → TradeTx → CauldronOutput2
  CauldronInput3 → TradeTx → CauldronOutput3
  UserBCHInput   → TradeTx → UserTokenOutput
                  TradeTx → UserBCHChangeOutput
```

No limit on how many Cauldron UTXOs a tx can interact with.

## Success requirements

1. **Re-create the contract** at the same output index as the input
2. **Constant Market Maker formula** must hold:
   - `kInput  = tokens * satoshis` of input
   - `kOutput = tokens * (satoshis - fee)` of output
   - `fee = |satoshisInput - satoshisOutput| * 0.03`
   - require: `kOutput >= kInput`

## Source URL
https://docs.riftenlabs.com/cauldron/swap