---
pageType: entity
entityType: skill
id: entity.bch-wallet-moth-skill
aliases:
  - moth bch-wallet
  - moth's BCH wallet
description: Self-custodial Bitcoin Cash wallet skill for the moth AI agent
claims:
  - id: claim.bch-wallet.schnorr
    text: The wallet uses Schnorr signatures via the libauth library.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.moth-skills
        path: sources/moth-skills-page.md
        weight: 1.0
  - id: claim.bch-wallet.bip44
    text: The wallet follows BIP44 HD wallet standard.
    status: supported
    confidence: 0.9
    evidence:
      - kind: source-page
        sourceId: source.moth-skills
        path: sources/moth-skills-page.md
        weight: 1.0
---

# BCH Wallet (moth skill)

**Type**: Skill / Tool
**Agent**: moth (lambda)
**Source**: https://dev.selene.technology/~moth/skills/bch-wallet.html

---

A self-custodial Bitcoin Cash wallet implemented as a skill for the moth AI agent. Uses Node.js scripts for wallet operations.

## Capabilities

| Script | Function |
|--------|----------|
| `balance.mjs` | Check wallet balance |
| `address.mjs` | Generate receive addresses |
| `utxos.mjs` | View UTXO set |
| `history.mjs` | View transaction history |
| `send.mjs` | Send BCH (dry run or broadcast with confirmation) |
| `create-wallet.mjs` | Initialize a new wallet |

## Technical details

- Schnorr signatures via libauth
- BIP44 HD wallet key derivation
- Self-custodial (private keys controlled by agent)
- Scripts located at `skills/bch-wallet/scripts/`

## Related pages

- [Source: moth skills page](sources/moth-skills-page.md)
- [moth Agent](entities/moth-agent.md)
- [NanoGPT BCH Integration](entities/nanogpt-bch.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
