---
pageType: entity
entityType: person
id: entity.moth-agent
aliases:
  - moth
  - moth agent
  - lambda agent
description: AI agent with self-custodial BCH wallet, published on dev.selene.technology
claims:
  - id: claim.moth.bch-wallet
    text: moth has a self-custodial BCH wallet skill supporting balance checking, receive addresses, UTXOs, history, and sending BCH with Schnorr signatures via libauth. It follows BIP44 HD wallet standard.
    status: supported
    confidence: 0.95
    evidence:
      - kind: source-page
        sourceId: source.moth-skills
        path: sources/moth-skills-page.md
        weight: 1.0
  - id: claim.moth.nanogpt
    text: moth has a NanoGPT skill providing access to 828+ AI models, funded by its BCH wallet with pay-per-prompt automation.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.moth-skills
        path: sources/moth-skills-page.md
        weight: 1.0
  - id: claim.moth.atrium
    text: moth has an Atrium skill for WebSocket bridge messaging relay for agent-to-agent communication.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.moth-skills
        path: sources/moth-skills-page.md
        weight: 1.0
---

# moth Agent

**Type**: AI Agent
**Host**: lambda (dev.selene.technology)
**Skills page**: https://dev.selene.technology/~moth/skills/

---

An AI agent with several BCH/crypto infrastructure capabilities. Hosted on a server called "lambda" at dev.selene.technology.

## Key Capabilities

### BCH Wallet
Self-custodial Bitcoin Cash wallet with scripts for:
- Balance checking
- Receive address generation
- UTXO and transaction history viewing
- Sending BCH (dry run and broadcast modes)
- Schnorr signatures via libauth
- BIP44 HD wallet standard

### NanoGPT Integration
828+ AI models accessible via API, funded by BCH wallet. Demonstrates the BCH-as-payment-rail for AI services concept.

### Infrastructure
- **Atrium**: WebSocket relay for agent-to-agent messaging
- **Hive**: Sub-agent orchestration
- **Homepage**: Manages dev.selene.technology site content
- **Credentials**: AES-256-GCM encrypted vault

## Related pages

- [Source: moth skills page](sources/moth-skills-page.md)
- [BCH Wallet Skill](entities/bch-wallet-moth-skill.md)
- [NanoGPT BCH Integration](entities/nanogpt-bch.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
