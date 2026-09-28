---
pageType: entity
entityType: nft-collection
id: entity.brainrot-collection
aliases:
  - Brainrot BCH
  - Brainrot NFT
  - BCH Brainrot
status: planned
description: Planned NFT collection of brainrot meme characters on Bitcoin Cash. Pokemon-style card game utility. Airdrop target: Broach (ROACH) holders.
related:
  - entities/broach-token
  - concepts/brainrot-types
  - syntheses/brainrot-bch-plan
---

# Brainrot Battles Collection

**Type**: NFT collection (CashTokens NFT, CashScript mega-contract)
**Official name**: **Brainrot Battles**
**Status**: Planning phase (Phase 0) — decisions locked 2026-07-12
**Plan**: `../syntheses/brainrot-bch-plan.md`

---

## What it is

A collection of **50,000 brainrot-meme NFTs** on Bitcoin Cash, distributed first as an airdrop to Broach ($ROACH) holders, then sold publicly via BCH, then made playable in a simple Pokémon-style card game of the same name ("Brainrot Battles").

## v1 Genesis roster (20 characters, 50,000 NFTs)

See the master plan for the full roster table. Quick stats:

- **6 types**: Aqua, Inferno, Verdant, Tempo, Espresso, Cosmic (Frost folded into Verdant; type chart in `concepts/brainrot-types.md`)
- **4 rarity tiers**: Mythic (5 chars × 250), Legendary (5 × 1,500), Rare (5 × 4,200), Common (5 × 4,050)
- **Total supply**: 1,250 + 7,500 + 21,000 + 20,250 = **50,000 NFTs exactly**
- **Mythic roster (revised 2026-07-12 10:56)**: Tralalero Tralala, Bombardiro Crocodilo, Tung Tung Tung Sahur, Ballerina Cappuccina, U Din Din Din. (La Vaca + Meowl moved to Legendary per Luke.)
- **Per-NFT commitment**: stores the character index (0-19) on-chain so the card game can read stats from wallet without an off-chain DB
- **Contract architecture**: one mega-contract (single CashTokens NFT category), character stored in `commitment` field

## Distribution

| Phase | Action | Cost to user |
|---|---|---|
| Airdrop | Direct mint to all ROACH holders (>0 balance, dust-filtered, CEX-denied) | Free |
| Allowlist | ROACH holders who didn't get airdrop, or anyone who wants first dibs | 30% off public |
| Public mint | Open until sold out | 0.001-0.10 BCH per NFT |

## Utility

- **v1**: PvE gauntlet (10 CPU battles), earn random Common NFTs + ROACH
- **v2**: Async PvP, ROACH prize pools, ranked mode
- **Long-term**: Burn NFTs for evolution, ROACH-as-stake for ranked, NFTs grant Discord roles

## Tech stack

| Layer | Choice |
|---|---|
| NFT standard | CashTokens NFT (CashScript) |
| Minting contract | `cashninjas/minting-contract` pattern |
| Metadata storage | IPFS (pinata or local node) |
| Metadata registry | BCMR (Bitcoin Cash Metadata Registry) via `indexer.cauldron.quest` |
| DEX venue | Cauldron (secondary sales) |
| Game frontend | Flutter Web → Flutter mobile |
| Wallet integration | Paytaca (mobile), Electron Cash / Paytaca (desktop), any CashScript-compatible BCH wallet |

## Open followups

- [ ] Confirm collection name (Luke decides)
- [ ] Confirm total supply (5k / 17k / 50k)
- [ ] Confirm contract architecture (per-character vs mega)
- [ ] Art direction (AI vs commissioned)
- [ ] BCMR auth from Cauldron team
- [ ] Broach team blessing (their holders list = our distribution)
- [ ] Marketing channel (TikTok brainrot creators)

## Related

- [Master plan](../syntheses/brainrot-bch-plan.md)
- [Broach token](broach-token.md) — airdrop target
- [Brainrot types chart](../concepts/brainrot-types.md)
- [Cauldron DEX](cauldron-dex.md)
- [CashTokens concept](../concepts/cash-tokens.md)