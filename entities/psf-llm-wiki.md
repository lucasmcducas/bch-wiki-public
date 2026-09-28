---
pageType: entity
entityType: project
id: entity.psf-llm-wiki
aliases:
  - PSF LLM Wiki
  - psf-llm-wiki
  - FullStack-Agents psf-llm-wiki
sourceUrl: https://github.com/FullStack-Agents/psf-llm-wiki
description: LLM Wiki about Bitcoin Cash, PSF, and the Cash Stack
claims:
  - id: claim.psf-llm-wiki.content
    text: The PSF LLM Wiki contains 120+ markdown pages covering BCH network, transactions, addresses, wallets, SLP tokens, CashTokens, PSF, mining, blockchain structure, and more.
    status: supported
    confidence: 0.95
    evidence:
      - kind: github-repo-contents
        sourceId: source.psf-llm-wiki
        path: sources/psf-llm-wiki.md
        weight: 1.0
  - id: claim.psf-llm-wiki.pattern
    text: The wiki follows Andrej Karpathy's LLM Wiki pattern with wiki/index.md table of contents, wiki/log.md operation log, and an AGENTS.md with maintenance instructions.
    status: supported
    confidence: 0.95
    evidence:
      - kind: direct-read
        sourceId: source.psf-llm-wiki
        path: sources/psf-llm-wiki.md
        weight: 1.0
  - id: claim.psf-llm-wiki.release
    text: Latest release is v3.0.1 (April 2026).
    status: supported
    confidence: 0.9
    evidence:
      - kind: github-release
        sourceId: source.psf-llm-wiki
        weight: 0.8
---

# PSF LLM Wiki

**Type**: GitHub Repository
**URL**: https://github.com/FullStack-Agents/psf-llm-wiki
**Owner**: FullStack-Agents
**Latest release**: v3.0.1 (April 2026)

---

A comprehensive LLM Wiki covering Bitcoin Cash (BCH), the Permissionless Software Foundation (PSF), and the Cash Stack. Based on Andrej Karpathy's LLM Wiki concept where an AI agent maintains structured knowledge from curated source documents.

## Topics covered by the wiki

- Bitcoin Cash network and protocol fundamentals
- Transactions (anatomy, validation, signing, chaining, broadcasting)
- Addresses (cashaddr, base58check, P2SH)
- Wallets (digital wallets, cold storage, minimal SLP wallet, BCH consumer wallets)
- SLP Tokens (Simple Ledger Protocol, fungible tokens, NFT1, genesis, mint, send, token indexer)
- CashTokens (specification, alternatives, rationale, stakeholders)
- PSF (Permissionless Software Foundation, PSF BCH API, PSF token)
- Mining and Proof of Work
- Blockchain data structure (blocks, headers, merkle tree, UTXO)
- Mempool and transaction pools
- CashScript (language, SDK, security, multi-contract)
- Infrastructure (BCHN full node, Fulcrum indexer, Cauldron API, IPFS integration)
- Layer protocols (X402 BCH payments, SLP Postage Protocol, Cash Stack layers)

## Related pages

- [Source: PSF LLM Wiki](sources/psf-llm-wiki.md)
- [Permissionless Software Foundation](entities/permissionless-software-foundation.md)
- [Cash Stack](concepts/cash-stack.md)
- [LLM Wiki Pattern](concepts/llm-wiki-pattern.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
