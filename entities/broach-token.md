---
pageType: entity
entityType: cash-token-fungible
id: entity.broach-token
aliases:
  - Broach
  - ROACH
  - $roach
sourceUrl: https://cashtokenmarkets.bch-1.org/api/tokens/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
description: Fungible CashToken on BCH. The "cockroach" BCH meme token. Live on Cauldron DEX. Planned airdrop target for the Brainrot BCH NFT collection.
claims:
  - id: claim.broach.basic
    text: Broach (ROACH) is a fungible CashToken on BCH, contract id 892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53, decimals 2, total supply 1,000,000,000 ROACH.
    status: supported
    confidence: 0.99
    evidence:
      - kind: api
        url: https://cashtokenmarkets.bch-1.org/api/tokens/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
        fetchedAt: 2026-07-12
        weight: 1.0
  - id: claim.broach.cauldron-pool
    text: Broach has a live Cauldron AMM pool at app.cauldron.quest/swap/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
    status: supported
    confidence: 0.95
    evidence:
      - kind: web
        url: https://app.cauldron.quest/swap/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
        fetchedAt: 2026-07-12
        weight: 0.95
  - id: claim.broach.meme
    text: Broach is the BCH community "cockroach" meme token — its description reads "A token epitomizing how relentless and hard to kill BCH is."
    status: supported
    confidence: 0.95
    evidence:
      - kind: api
        url: https://cashtokenmarkets.bch-1.org/api/tokens/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
        fetchedAt: 2026-07-12
        weight: 1.0
  - id: claim.broach.no-native-nft
    text: Broach is a pure-ft CashToken category (isFungible=true, hasNFT=false). It does not have a native NFT capability; the planned Brainrot BCH NFT collection will be a separate category.
    status: supported
    confidence: 0.95
    evidence:
      - kind: api
        url: https://cashtokenmarkets.bch-1.org/api/tokens/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53
        fetchedAt: 2026-07-12
        weight: 1.0
  - id: claim.broach.holders-unknown
    text: Actual holder count, top holder distribution, and pool TVL for Broach are not exposed by the cashtokenmarkets per-token API as of 2026-07-12. Will need to query indexer.cauldron.quest directly for the airdrop snapshot.
    status: supported
    confidence: 0.85
    evidence:
      - kind: api-experiment
        url: https://cashtokenmarkets.bch-1.org/api/holders, https://indexer.cauldron.quest/...
        fetchedAt: 2026-07-12
        weight: 0.85
---

# Broach (ROACH)

**Type**: Fungible CashToken (BCH)
**Contract ID (categoryId)**: `892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53`
**Decimals**: 2
**Total supply**: 1,000,000,000 ROACH (100,000,000,000 base units)
**Created**: 2025-11-25
**Cauldron pool**: [Live](https://app.cauldron.quest/swap/892cef80a326f92583766abb85562f550fe6a6ff9a7d458067680c720135ff53)
**Status**: Live

---

## Description

> *"A token epitomizing how relentless and hard to kill BCH is."*

Broach is the BCH community's cockroach meme token. It's a BCH-native (CashToken) fungible token with a Cauldron AMM pool. The cockroach framing is a self-aware joke about BCH's reputation as the chain that just won't die — and that, after every "flippening" prediction, is still here.

## Facts (verified live 2026-07-12)

Pulled via `https://cashtokenmarkets.bch-1.org/api/tokens/892cef80…`:

| Field | Value |
|---|---|
| Name | Broach |
| Symbol | ROACH |
| Decimals | 2 |
| Total supply (base) | 100,000,000,000 |
| Total supply (display) | 1,000,000,000 ROACH |
| isFungible | true |
| hasNFT | false |
| iconUrl | `https://ipfs.tapswap.cash/ipfs/QmbAF345YoHhSzeGpvQEVo52DKkcu7mkcEvfuFxr8haqA9` |
| Created | 2025-11-25T17:14:20.399Z |
| Registry | `https://indexer.cauldron.quest` |
| Last registry sync | 2026-02-26T21:00:14.687Z |

## Holder data — NOT YET KNOWN

Cashtokenmarkets' per-token API does not return `holderCount` for Broach (the field exists in `/api/tokens?limit=200` but is `null` for Broach). Cauldron's indexer endpoints (`/api/tokens/{id}/holders`, `/api/holders/{id}`) all 404 as of 2026-07-12.

**Next step:** Query `indexer.cauldron.quest` directly via GraphQL or REST once we have auth credentials (likely a BCMR auth token). Alternative: use `bitquery.io` GraphQL for BCH.

## Why we picked Broach for the airdrop

- **Active holder base** = warm distribution. The whole point of the Brainrot Battles project is "airdrop to existing ROACH holders first, then public sale."
- **Cockroach framing = on-brand** for a meme NFT collection about resilient characters.
- **Has a Cauldron pool** = liquidity for in-game ROACH rewards.
- **No NFT conflict** = clean slate for our NFT category.

## Treasury wallet plan

Per Luke's decision (2026-07-12), the **broach deployer wallet will also serve as the Brainrot Battles treasury** if doable. Implications:

- ✅ Single key to manage = simple ops
- ✅ CashScript `mintingContract` owner pubkey = same pubkey = airdrop + public mint + payout all flow to one wallet
- ⚠️ Shared risk: if broach OR brainrot is compromised, both are
- 📝 Flagged for upgrade to 2-of-3 multisig in v2 (after launch)

**Need from Luke:** verify he has the broach deployer seed/keys and confirm the wallet pubkey before we deploy the Brainrot Battles contract.

## Related

- [Brainrot BCH plan](../syntheses/brainrot-bch-plan.md) — the project this airdrop feeds into
- [Brainrot collection](../entities/brainrot-collection.md) — the NFTs to be airdropped
- [Cauldron DEX](cauldron-dex.md) — the DEX venue for ROACH and the NFT minting
- [CashTokens concept](../concepts/cash-tokens.md) — the underlying token standard