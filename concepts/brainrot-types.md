---
pageType: concept
id: concept.brainrot-types
description: 6-type Pokemon-style elemental matchup chart used by the Brainrot BCH card game
related:
  - entities/brainrot-collection
  - syntheses/brainrot-bch-plan
---

# Brainrot Type Chart

The Brainrot BCH card game uses **6 types** (cleaner than Pokémon's 18, deeper than 3). Damage multipliers are **2× super effective, 0.5× not very effective**.

## The six types

| Type | Color | Emoji | Theme | Brainrot characters |
|---|---|---|---|---|
| **Aqua** | Blue | 🌊 | Water, sea creatures | Tralalero Tralala, Burbaloni Luliloli |
| **Inferno** | Red | 🔥 | Fire, bombs, aircraft | Bombardiro Crocodilo, Bombombini Gusini |
| **Verdant** | Green | 🌵 | Plants, fruits, nature | Lirili Larila, Chimpanzini Bananini, Graipussi Medussi |
| **Tempo** | Purple | 🥁 | Music, percussion, rhythm | Tung Tung Tung Sahur, Brr Brr Patapim, Bri Bri Bicus Dicus |
| **Espresso** | Brown | ☕ | Coffee, food, beverages | Ballerina Cappuccina, Cappuccino Assassino, Espressona Signora |
| **Cosmic** | Magenta | 🪐 | Space, mystic, planetary | La Vaca Saturno Saturnita, U Din Din Din |

*Beast was originally an option but got folded into Verdant — keeps the chart cleaner.*

## Matchup table

`attacker row × defender column = damage multiplier`

|         | Aqua | Inferno | Verdant | Tempo | Espresso | Cosmic |
|---------|------|---------|---------|-------|----------|--------|
| **Aqua**    | 1× | **2×** | **0.5×** | 1× | 1× | **0.5×** |
| **Inferno** | **0.5×** | 1× | **2×** | 1× | 1× | 1× |
| **Verdant** | **2×** | **0.5×** | 1× | **0.5×** | **2×** | 1× |
| **Tempo**   | 1× | 1× | **2×** | 1× | **0.5×** | **2×** |
| **Espresso**| 1× | 1× | **0.5×** | **2×** | 1× | 1× |
| **Cosmic**  | **2×** | 1× | 1× | **0.5×** | 1× | 1× |

### Symmetry check

Every 2× has a matching 0.5× in the same column → consistent. Examples:
- Aqua beats Inferno (2×) → Inferno weak to Aqua (0.5×) ✓
- Verdant beats Aqua (2×) → Aqua weak to Verdant (0.5×) ✓
- Tempo beats Verdant (2×) → Verdant weak to Tempo (0.5×) ✓
- Espresso beats Tempo (2×) → Tempo weak to Espresso (0.5×) ✓
- Cosmic beats Aqua (2×) → Aqua weak to Cosmic (0.5×) ✓
- Cosmic beats Tempo (2×) → Tempo weak to Cosmic (0.5×) ✓

### Type triangle (the rock-paper-scissors core)

```
Aqua      → beats → Inferno  → beats → Verdant  → beats → Aqua
Tempo     → beats → Verdant  → beats → ???       → beats → Tempo
Espresso  → beats → Tempo    → beats → Verdant   → beats → Espresso
Cosmic    → beats → Aqua & Tempo
```

There's no single dominant type — Tempo, Espresso, and Cosmic all have overlapping strengths. This forces players to think about team composition, not just stack one type.

## Stat modifiers by type

| Type | HP | ATK | DEF | SPD | Special |
|---|---|---|---|---|---|
| Aqua | 100% | 100% | 100% | 110% | — |
| Inferno | 90% | 120% | 80% | 110% | — |
| Verdant | 120% | 90% | 110% | 80% | — |
| Tempo | 100% | 100% | 90% | 120% | 10% dodge |
| Espresso | 90% | 115% | 90% | 100% | 15% crit |
| Cosmic | 100% | 95% | 120% | 85% | 20% status resist |

Modifiers apply on top of the per-character base stats. Average ATK × HP is roughly equal across types → matchups matter more than raw stats.

## Ability design space (per type)

Each character gets one unique ability, themed to their type:
- **Aqua**: evasion, heal
- **Inferno**: high-damage nukes, self-damage recoil
- **Verdant**: regen, status effects (poison, sleep)
- **Tempo**: speed manipulation, multi-hit
- **Espresso**: crit-bursts, instant-KO under conditions
- **Cosmic**: stat-swap, defense pierce, status immunity

This gives 20 characters × 6 ability archetypes = plenty of design space without overlap.

## Per-collection allocation (Brainrot Battles 50k, revised 2026-07-12 10:56)

| Type | Characters | Total supply | % of collection |
|---|---|---|---|
| **Aqua** | Tralalero (Mythic), Burbaloni (Legendary) | 1,750 | 3.5% |
| **Inferno** | Bombardiro (Mythic), Bombombini (Legendary) | 1,750 | 3.5% |
| **Verdant** | Meowl (Legendary), Lirili (Legendary), Bobrito/Trippi/Graipussi (Rare), Chimpanzini/Boneca (Common) | 23,700 | 47.4% |
| **Tempo** | Tung Tung (Mythic), Brr Brr (Rare), Bri Bri (Common) | 8,500 | 17.0% |
| **Espresso** | Ballerina (Mythic), Cappuccino (Rare), Espressona (Common) | 8,500 | 17.0% |
| **Cosmic** | U Din Din (Mythic), La Vaca (Legendary) | 1,750 | 3.5% |
| **Frost** | Frigo Camelo (Common) | 4,050 | 8.1% |

**Verdant is heavy** (47%) — reflects the brainrot source roster (lots of plant/animal hybrids). Game balance still works because type matchups matter more than raw supply. Will rebalance in v2 if needed.

**No Verdant Mythic** as of 2026-07-12 10:56 revision (Meowl demoted to Legendary). Mythics now cover Aqua, Inferno, Tempo, Espresso, Cosmic — only Frost has no Mythic, but Frost is a single-character type anyway.

## Related

- [Brainrot Battles master plan](../syntheses/brainrot-bch-plan.md)
- [Brainrot Battles collection](../entities/brainrot-collection.md)