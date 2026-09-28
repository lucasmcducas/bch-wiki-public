<!-- openclaw:wiki:raw-source -->
---
pageType: source
sourceType: gitlab
sourceUrl: https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node
title: Bitcoin Cash Node — GitLab repository metadata
accessedAt: 2026-09-14
description: Project metadata, languages, releases from GitLab API for bitcoin-cash-node/bitcoin-cash-node
---

# Source: Bitcoin Cash Node (GitLab repo metadata)

**Source**: https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node
**Project ID**: 16979301
**Path**: `bitcoin-cash-node/bitcoin-cash-node`
**Default branch**: `master`
**Visibility**: public
**License**: None (project-level; check `COPYING` and per-file headers — historically MIT-style)
**Created**: 2020-02-18 (forked from Bitcoin ABC during the IFP split)
**Last activity**: 2026-09-14

## Description (from GitLab)

> A professional, miner-friendly node that solves practical problems for Bitcoin Cash.
> https://bitcoincashnode.org

## Languages (GitLab Languages API)

```json
{
  "C++": 70.49,
  "Python": 17.24,
  "C": 7.02,
  "CMake": 1.38,
  "Shell": 1.17
}
```

## Releases (most recent first)

| Tag | Released | Headline |
|---|---|---|
| v29.1.0 | 2026-07-28 | RPC improvements, perf optimizations, byteCodePatterns on gettxout |
| v29.0.0 | 2026-01-09 | May 15, 2026 network upgrade (P2S, loops, functions, bitwise) |
| v28.0.1 | 2024-12-29 | Bug fixes |
| v28.0.0 | 2024-11-30 | May 15, 2025 upgrade (VM limits, BigInt) |
| v27.1.0 | 2024-07-10 | Maintenance |

## Counts

- Forks: 154
- Stars: 83
- Open issues: (not exposed via GitLab API for this project)

## Notes

- The "License: None" field on GitLab means the maintainers did not set a top-level `LICENSE` metadata key. The actual source uses MIT-style headers per file; verify before commercial reuse.
- Python 17% is overwhelmingly functional tests under `test/functional/`, not application code. The C++ 70% is the production codebase.
- Mirror on GitHub: https://github.com/bitcoin-cash-node/bitcoin-cash-node (used for some release-note pages and tagged releases)

## Related

- [Bitcoin Cash Node entity](entities/bchn-node.md)
- [CashTokens concept](concepts/cash-tokens.md)
