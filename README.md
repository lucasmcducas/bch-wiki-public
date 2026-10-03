# bch-wiki-public

BCH × Omarchy knowledge base — entities, references, security knowledge base.

This is the shared brain for community LLMs working on the BCH × Omarchy plugin
collaboration (see [`syntheses/bch-omarchy-collaboration-plan.md`](syntheses/bch-omarchy-collaboration-plan.md)).

## Companion repos

- [lucasmcducas/bch-bot-omarchy](https://github.com/lucasmcducas/bch-bot-omarchy) — the Omarchy plugin
- [lucasmcducas/bch-bot](https://github.com/lucasmcducas/bch-bot) — the underlying wallet CLI (private; community members can mirror the public subset)

## Contents

| Directory | What |
|---|---|
| `entities/` | Reference entries on BCH ecosystem projects (Cauldron DEX, libauth, PUSD, ROACH, etc.) |
| `references/` | Topical reference docs (zero-conf security, economics primer, Omarchy research) |
| `references/swaps/` | **Swaps on a CashTokens AMM** — how a Cauldron pool is really represented on-chain, the k-invariant a route must satisfy, and the class of bug where a safety check reports success on a dead route |
| `sources/` | Raw source excerpts (release notes, official docs) |
| `concepts/` | BCH concepts (CashTokens, Cash Stack, etc.) |
| `security/` | BCH UTXO + wallet security knowledge base (6 docs) |
| `syntheses/` | Higher-level synthesis docs (architecture, PUSD integration, the collaboration plan itself) |

## What's NOT here

The private [`lucasmcducas/ai-workspace-backup`](https://github.com/lucasmcducas/ai-workspace-backup) repo contains additional syntheses — internal planning docs — that are specific to the wallet maintainer's decisions and not relevant to community contributors.

## Audit + safety

Run `./audit-public.sh` before each push. The script fails the commit if any
forbidden strings leak through (treasury addresses, mainnet wallet paths,
specific dollar figures, etc.).

## License

MIT — see [LICENSE](LICENSE).
