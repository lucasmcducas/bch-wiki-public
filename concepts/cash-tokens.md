---
pageType: concept
id: concept.cash-tokens
aliases:
  - CashTokens
  - CHIP tokens
sourceUrl: https://gitlab.com/GeneralProtocols/cashtokens
description: Modern fungible and non-fungible token protocol native to Bitcoin Cash
claims:
  - id: claim.cash-tokens.modern-standard
    text: CashTokens is the current and modern token protocol on Bitcoin Cash, native at the consensus level via CHIP (Cash Improvement Proposal).
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.bch-knowledge-base
        path: sources/bch-knowledge-base.md
        weight: 0.9
  - id: claim.slp-deprecated
    text: SLP (Simple Ledger Protocol) is effectively deprecated/discontinued. CashTokens is the successor and modern token standard on BCH.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.bch-knowledge-base
        path: sources/bch-knowledge-base.md
        weight: 0.8
---

# CashTokens

**CashTokens** is the modern token protocol for Bitcoin Cash, native at the consensus level. It was activated via the May 2023 BCH upgrade (CHIP: Cash Improvement Proposal) and is the successor to the now-deprecated SLP (Simple Ledger Protocol).

## Key Characteristics

- **Native protocol** — Tokens are first-class citizens in the BCH protocol, not an overlay like SLP. Transactions with tokens use standard BCH transaction infrastructure.
- **Fungible tokens** — For currencies, stablecoins, utility tokens, governance tokens
- **Non-fungible tokens (NFTs)** — Unique digital assets
- **Low fees** — Token transactions cost the same as regular BCH transactions (~$0.001)
- **CashScript compatible** — Can be used with smart contracts via CashScript

## Why CashTokens Replaced SLP

- SLP was an overlay protocol (token state tracked independently from the base chain) — required separate indexers and had failure modes where the overlay could diverge from the chain
- CashTokens is **consensus-enforced** — token validity is guaranteed by BCH miners and full nodes
- SLP maintainers have largely stopped development; the ecosystem has migrated to CashTokens

## Use Cases

- Tokenized assets (real estate, commodities, hardware shares)
- Utility tokens (compute credits, prepaid services)
- Community currencies and stablecoins
- NFTs on BCH

## Related pages

- [Cash Stack](concepts/cash-stack.md)
- [BCH Knowledge Base](entities/bch-knowledge-base.md)
- [Permissionless Software Foundation](entities/permissionless-software-foundation.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
