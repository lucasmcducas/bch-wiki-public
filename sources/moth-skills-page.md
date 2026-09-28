---
pageType: source
sourceType: web
sourceUrl: https://dev.selene.technology/~moth/skills/
title: moth skills index
accessedAt: 2026-07-01
description: Skills page for the "moth" AI agent, hosted on dev.selene.technology
---

# moth Skills Index

**Source**: https://dev.selene.technology/~moth/skills/

**Accessed**: 2026-07-01

---

moth is an AI agent running on a server called "lambda". Its skills page lists capabilities relevant to BCH/crypto infrastructure.

## Key BCH-relevant skills

### bch-wallet
Self-custodial Bitcoin Cash wallet. Supports:
- Balance checking (via `balance.mjs`)
- Receive address generation (via `address.mjs`)
- UTXO and history viewing (via `utxos.mjs`, `history.mjs`)
- Sending BCH with Schnorr signatures via libauth (dry run and broadcast modes)
- BIP44 HD wallet
- Create wallet via `create-wallet.mjs`

### nanogpt
Access to 828+ AI models via NanoGPT API, funded by BCH wallet. Pay-per-prompt with full automation. Direct integration between BCH payments and AI model access.

### Other infrastructure skills
- **atrium** — WebSocket bridge relay for agent-to-agent communication
- **hive** — Sub-agent orchestration
- **credentials** — Encrypted credential vault (AES-256-GCM)
- **homepage** — Managing the dev.selene.technology site

## Related pages

- [bch-wallet](entities/bch-wallet-moth-skill.md)
- [moth agent](entities/moth-agent.md)
- [nanogpt](entities/nanogpt-bch.md)

## Related
<!-- openclaw:wiki:related:start -->
- No related pages yet.
<!-- openclaw:wiki:related:end -->
