---
pageType: synthesis
id: synthesis.bch-omarchy-collaboration-plan
description: Plan for BCH community members with idle compute to collaborate on the BCH × Omarchy plugin via git forks + PRs. Five scoped work units, contributor setup, risk model.
sourceUrl: bch-bot + bch-bot-omarchy repos
---

# BCH × Omarchy Plugin — Collaborative LLM Build Plan

> Companion to `syntheses/bch-bot-omarchy-monetization-plan.md` (the product plan).
> This doc is the *team plan*: how multiple humans + LLMs build it together.

## TL;DR

BCH community members with idle compute (each running their own LLM agent) collaborate on the BCH × Omarchy plugin via **git forks + pull requests**. Each contributor's LLM works one scoped GitHub issue → one PR → maintainer review → merge. Per-tx 0.5% treasury fee (configurable, user-visible).

## What's already done (the foundation)

| Repo | Path | What it has |
|---|---|---|
| **bch-bot** | `github.com/lucasmcducas/bch-bot` | Working BCH wallet — Node.js + libauth, 216 passing tests, treasury fee wired in, encrypted at rest, on `bch-bot/phase-1` branch |
| **bch-bot-omarchy-plugin** | `github.com/lucasmcducas/bch-bot-omarchy` (not pushed yet) | Initial Omarchy plugin scaffold — manifest, bar widget, IPC command, README, LICENSE, SECURITY.md. Branch: `main`. Head: `20ea2c9` |
| **memory-bch-wiki** | `github.com/lucasmcducas/ai-workspace-backup` | Knowledge base — Omarchy distro + marketplace entity docs, security KB (4 docs) on `security/kb-init` branch, monetization synthesis, capital accumulation strategy |

## The collaboration model

### Why forks + PRs beats shared monorepo

Git forks + PRs is the proven collaborative dev model. Each LLM works on a fork owned by the BCH community member; PRs flow back to `lucasmcducas/bch-bot-omarchy` for review and merge. Shared monorepos break when multiple LLMs `git push --force` over each other. Forks keep trust boundaries clean.

### Repository structure (target)

```
lucasmcducas/bch-bot-omarchy       ← upstream
├── main                            ← released versions only
├── collaborate/v0.1                ← integration branch for community PRs
├── collaborate/v0.1-bar-widget     ← topic branch
├── collaborate/v0.1-commands       ← topic branch
├── collaborate/v0.1-preview        ← topic branch
└── collaborate/v0.1-tests          ← topic branch

<community-member>/bch-bot-omarchy  ← each member's fork
└── feature/<name>                  ← their working branch
```

## Roles + responsibilities

| Role | Who | What they do |
|---|---|---|
| **Upstream maintainer** | Luke + Jav | Merge PRs, cut releases, manage marketplace submission |
| **Wiki steward** | Whoever wants to own it | Triage new wiki pages, resolve contradictions, prune stale |
| **Compute contributor** | BCH community members | Set up sandbox, approve LLM PRs before they go upstream, run on real Omarchy |
| **Reviewer** | Maintainers + trusted reviewers | Read diffs, run plugin in their own Omarchy, request changes |

### Why "compute contributors" is the right framing

A BCH community member with idle GPU/CPU isn't doing the work themselves — their LLM agent is. The human's role: (1) set up the sandbox, (2) approve the LLM's PRs before they go upstream, (3) run the resulting code on real Omarchy. Trust boundary stays clean: humans review, LLMs draft, humans approve.

## First-week plan

### Day 1: Set up shared infrastructure

1. Push `~/bch-bot-omarchy-plugin` to `github.com/lucasmcducas/bch-bot-omarchy`
2. Create `collaborate/v0.1` branch and push it
3. Add `COLLABORATION.md` + `CONTRIBUTING.md` to repo root
4. Add issue templates for the 5 scoped work units

### Day 2-3: Publish 5 scoped work units as GitHub issues

| # | Issue | Acceptance criteria |
|---|---|---|
| **#1** | Wire `bch-bot balance` into the bar widget | Bar shows live BCH balance updating every 60s; no network code in QML |
| **#2** | Send modal with treasury fee disclosure | Fee line visible; confirmation gates on `BCH_CONFIRM=yes` |
| **#3** | QR code generator for receive | QR encodes the address; works in shell command output |
| **#4** | preview.png + marketplace submission prep | Screenshot of bar widget, 1200x630, ≤50MB |
| **#5** | Submit the marketplace listing | Issue opened at `omacom/omarchy-plugin-marketplace` with all 5 checklist items |

### Day 4-7: First wave of contributions

Each community member picks one issue. LLM does the work. PRs flow in.

## Contributor setup — exact commands

```bash
# 1. Fork the repo on GitHub (one click)

# 2. Clone their fork
git clone https://github.com/<their-handle>/bch-bot-omarchy.git
cd bch-bot-omarchy
git remote add upstream https://github.com/lucasmcducas/bch-bot-omarchy.git
git fetch upstream
git checkout -b collaborate/v0.1 upstream/collaborate/v0.1
git checkout -b feature/<issue-number>-<short-name>

# 3. Point their LLM at:
#    - this MD file
#    - entities/omarchy-plugin-marketplace.md (manifest schema)
#    - SECURITY.md (the marketplace baseline)
#    - lib/wallet-encryption.mjs from bch-bot

# 4. LLM does the work, commits
git add -A
git commit -m "fix(#1): wire bch-bot balance into bar widget via Quickshell.Io"
git push origin feature/1-balance-display

# 5. Human reviews the diff, opens PR
#    base: lucasmcducas/bch-bot-omarchy collaborate/v0.1
#    head: <their-fork>:<branch>
```

### What the LLM needs to know (exact list)

1. **Plugin ID** is `io.github.lucasmcducas.bch-wallet` — do not change.
2. **Manifest schema** is `schemaVersion: 1` with `kinds`, `entryPoints`, optional kind-specific config block.
3. **Security baseline** blocks `curl|sh`, unpinned git exec, passwordless sudoers, bundled binaries, dangerous `/tmp` use. The plugin does none of these — LLM should not introduce any.
4. **Quickshell API** for IPC: `Quickshell.Io.Process` for spawning CLI; `Quickshell.Io.IpcHandler` for receiving commands. QML needs testing in real Omarchy to verify.
5. **Treasury fee** is `treasuryBps = 50` (0.5%) by default; visible to user; configurable to 0 or another address.
6. **No write access outside plugin scope** — do not modify `~/.config/omarchy/` outside the plugin's own settings.
7. **Test before PR** — `node scripts/test-*.mjs` style tests encouraged for any logic QML defers to.

## The wiki is the shared brain

The BCH wiki (`memory-bch-wiki` on `security/kb-init` branch) is what every LLM should read first. To make it discoverable:

1. **Mirror the wiki to a public repo** — currently `lucasmcducas/ai-workspace-backup` contains unrelated content. Extract the BCH-relevant subset into `lucasmcducas/bch-wiki` and make it public.
2. **Add a `BOOTSTRAP.md`** at the wiki repo root with:
   - 5-10 most important pages to read
   - The collaboration model + link to this plan
   - How to submit wiki changes (PR to wiki, not just commits)
3. **Reference the wiki from the plugin repo** — `README.md` should link to relevant entity pages.

## Risk model

| Risk | Likelihood | Mitigation |
|---|---|---|
| Two LLMs edit the same file, conflicts in PR | High | Work in different files when possible; one issue = one PR; small scope per PR |
| LLM introduces a security baseline violation | Medium | Maintainer review with baseline checklist; CI lint greps for `curl|sh` patterns |
| LLM leaks a wallet private key in a PR | Low | Plugin repo has no wallet. bch-bot repo has wallet code; tests use ephemeral keys. |
| Marketplace rejection after submission | Medium | Read SUBMISSION.md + SECURITY.md carefully before submitting; preview issue body before opening |
| Contributor diverges from per-tx fee model | Medium | This plan locks the model: 0.5% flat, treasury address in plain sight, user-overridable. PRs changing this need explicit maintainer approval |

## What "done" looks like for v0.1

- [ ] `bch-bot-omarchy` is on GitHub (public)
- [ ] `collaborate/v0.1` branch exists with scaffold
- [ ] `COLLABORATION.md` + `CONTRIBUTING.md` at repo root
- [ ] 5 GitHub issues published, one per work unit
- [ ] First PR from community member merged
- [ ] Plugin tested in real Omarchy 4.0 install
- [ ] `preview.png` generated
- [ ] Marketplace submission issue opened

After v0.1 ships, v0.2 candidates:
- PUSD stake widget
- Cauldron LP widget (real-time pool value)
- Multi-wallet support
- Hardware wallet integration (Ledger/Trezor via PSBT)
- Token NFT gallery (read-only, no signing)

## The 5 things to approve before execution

1. **Push `~/bch-bot-omarchy-plugin` to GitHub as `lucasmcducas/bch-bot-omarchy`** (creates public repo)
2. **Create `collaborate/v0.1` integration branch** and push COLLABORATION.md + CONTRIBUTING.md + issue templates
3. **Mirror the bch wiki to a public repo** (`lucasmcducas/bch-wiki`) with BOOTSTRAP.md
4. **Open the 5 GitHub issues** (one per work unit, with acceptance criteria)
5. **Send invite to BCH community members** with short message + this plan attached

Each is independently reversible.

## One-paragraph pitch for community members

> We're building the first spendable crypto plugin for the Omarchy Plugin Marketplace — a self-custodial Bitcoin Cash (BCH) wallet in your Omarchy bar. The wallet code (`bch-bot`) is at `github.com/lucasmcducas/bch-bot` (Node.js, 216 tests passing). The plugin scaffold is at `github.com/lucasmcducas/bch-bot-omarchy`. We need community help on 5 scoped work units (wire balance display, send modal with fee disclosure, QR generator, preview screenshot, marketplace submission). Each unit is one issue = one PR. Fork the repo, point your LLM at this plan + the wiki entities, review the diff, open the PR. Maintainers (Luke + Jav) review + merge. Per-tx 0.5% treasury fee (configurable, user-visible, transparently disclosed in the manifest). MIT-licensed, no vendor lock-in.
