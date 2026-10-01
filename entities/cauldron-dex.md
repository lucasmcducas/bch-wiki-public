---
pageType: entity
entityType: dex
id: entity.cauldron-dex
aliases:
  - Cauldron
  - cauldron.quest
  - Riften Labs Cauldron
sourceUrl: https://www.cauldron.quest
description: First AMM DEX on Bitcoin Cash CashTokens; flagship product of Riften Labs; uses micro-pools (one UTXO per pool)
claims:
  - id: claim.cauldron.fee
    text: Cauldron charges a 0.3% fee to liquidity providers on every trade.
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.cauldron-docs-swap
        path: sources/cauldron-docs-swap.md
        weight: 1.0
  - id: claim.cauldron.model
    text: Cauldron uses a Constant Product Market Maker (CPMM / x*y=k) model, identical to Uniswap V2, but every pool is its own independent contract/UTXO (micro-pool design).
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.cauldron-docs-swap
        path: sources/cauldron-docs-swap.md
        weight: 1.0
  - id: claim.cauldron.swap-mechanics
    text: A swap consumes a Cauldron UTXO as input and re-creates the contract in the same output index. Multiple Cauldron UTXOs can be used in one transaction for additional liquidity / better prices.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.cauldron-docs-swap
        path: sources/cauldron-docs-swap.md
        weight: 1.0
  - id: claim.cauldron.labs
    text: Cauldron is the flagship product of Riften Labs. Riften Labs is also behind Moria (the Liquity fork on BCH) and Delphi (oracle).
    status: supported
    confidence: 0.85
    evidence:
      - kind: source-page
        sourceId: source.cauldron-docs
        path: sources/cauldron-docs.md
        weight: 0.9
---

# Cauldron DEX

**Type**: AMM DEX (CPMM / x*y=k)
**Website**: https://www.cauldron.quest
**Operator**: Riften Labs
**Launched**: ~2024 (first DEX on CashTokens)
**Status**: Live

---

## What it is

Cauldron is the first AMM DEX built on BCH's CashTokens. It lets anyone swap or provide liquidity for any CashToken. It is the de facto liquidity venue for PUSD/BCH and the entire BCH DeFi ecosystem.

## Architecture: micro-pools

Unlike Uniswap V2 (one big factory contract holding all reserves), **each Cauldron pool is its own independent contract and its own UTXO**. Every LP deposit creates a new UTXO; every swap consumes and re-creates the contract at the same output index.

Consequences:
- ✅ Anyone can deploy any pair — no permissioning, no factory gating
- ✅ No "router" contract hack can drain all pools
- ✅ Multiple pools can be aggregated in a single transaction for better prices
- ⚠️ TVL is fragmented across many small pools
- ⚠️ Indexer is critical (Riften Labs runs one) — wallets need it to find liquidity

## Fee structure

- **0.3% per trade**, all goes to LPs
- No protocol fee / no treasury cut from Cauldron itself
- LPs earn pure swap fees plus any extra incentive rewards programs

## Swap mechanics (from docs)

A simple BCH → token swap looks like:
```
CauldronInput → TradeTx → CauldronOutput
UserBCHInput → TradeTx → UserTokenOutput
                TradeTx → UserBCHChangeOutput
```

The constant-product invariant must hold:
```
kInput  = tokensIn * satoshisIn
kOutput = tokensOut * (satoshisOut - fee)
fee     = |satoshisIn - satoshisOut| * 0.03
require kOutput >= kInput
```

Multi-pool swaps just chain more Cauldron inputs/outputs in one tx.

## Surrounding ecosystem (Riften Labs)

| Product | Function |
|---------|----------|
| Cauldron | AMM DEX |
| **Moria** | Liquity-style stablecoin / borrowing (used by some BCH stablecoin teams) |
| Delphi | Price oracle for BCH/CashTokens |
| WizardConnect | Wallet connection protocol (xpub-based) |
| Moria v0 | Earlier test version |

## Strong points
- Battle-tested CPMM (Uniswap V2 math)
- Non-custodial, no KYC, no censorship
- Anyone can create a pool for any CashToken
- Cheap (BCH tx fees, <$0.001)

## Weak points / frictions for our purposes
- Web-only, no mobile app (significant gap for consumer users)
- No fiat on-ramp
- No "earn" tab — you have to know what LPing means
- Impermanent loss is real and unexplained to normies
- No auto-compounding of LP fees
- TVL is tiny vs. EVM DEXes → slippage on big trades
- No limit orders, no perps, no structured products
- 0.3% fee is standard but high for stablecoin swaps (where Curve/Uniswap V3 use 0.01–0.05%)

## Who can actually execute a swap

A swap spends a Cauldron pool UTXO, and that input is committed to the **pool
operator's** public key hash in the pool's locking bytecode. No user wallet
holds that key, so a wallet alone cannot produce a valid swap — this is a
property of the protocol, not a wallet limitation.

[Riften Labs](riften-router.md) (Cauldron's operator, and the organisation
behind Delphi and Moria) run the **Cauldron Router**, a public no-auth
WebSocket service that assembles the unsigned transaction and names which
inputs the client owns. The wallet signs only those, and broadcasts via
`https://broadcast.cauldron.quest/broadcast`.

Practical cost of that route: the router adds a fee on top of the 0.3% LP fee
(10 bps / 0.1% at time of writing — read it from the build response, it may
change). See [Riften Labs Cauldron Router](riften-router.md) for the protocol
and [BCH signing and verification](../references/bch-signing-and-verification.md)
for the partial-signing traps it introduces.

## Related pages
- [Riften Labs Cauldron Router](riften-router.md) — the service that makes a swap executable
- [ParyonUSD](entities/paryonusd.md) — biggest CashToken by usage
- [CashTokens](concepts/cash-tokens.md)
- [Source: Cauldron docs](sources/cauldron-docs.md)
- [Source: Cauldron swap mechanics](sources/cauldron-docs-swap.md)