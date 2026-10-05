---
pageType: entity
id: entity.omarchy
description: Omarchy — opinionated Arch + Hyprland + Quickshell distro by DHH; potential distribution channel for the bch-bot wallet with per-tx monetization. Research note: references/omarchy-research.md.
sourceUrl: https://omarchy.org
---

# Omarchy

## What it is

Omarchy is an open-source Linux distribution created by **David Heinemeier Hansson (DHH)** — the creator of Ruby on Rails and co-owner of 37signals — with Ryan Hughes as co-developer. Released June 26, 2025, initially as an opinionated post-install of Arch Linux + Hyprland. Current major release: **4.0 "Quattro"** (August 14, 2026), which replaced several Wayland components (Waybar, Walker, Mako, SwayOSD, hyprlock, polkit-gnome) with a unified **Quickshell** desktop shell.

The naming is "**omakase**" (お任せ) — "leave it to the chef." Direct successor to **Omakub** (DHH's earlier Ubuntu-based developer setup). MIT-licensed core.

## Backing (Aug 2026+)

The **Omacom Foundation** (nonprofit) funds continued development. Notable donors include Brian Armstrong (Coinbase CEO), Patrick Collison (Stripe), Michael Dell, Jack Dorsey (Block), Drew Houston (Dropbox), Tobias Lütke (Shopify), Peter Steinberger (OpenClaw), 1Password, 37signals. Reported total ≈ $12M+; DigitalOcean added $3M in Sept 2026. **Brian Armstrong being a donor is directly relevant — Coinbase ships a read-only plugin in the Omarchy marketplace.**

## Architecture (quick reference)

- Base: Arch Linux (rolling)
- Package manager: pacman
- Own package repo: **OPR** (Omarchy Package Repository; multi-channel edge→rc→stable, multi-arch; `omacom/omarchy-pkgs`)
- Init: systemd
- Display: Wayland only
- Compositor: Hyprland
- Desktop shell: Quickshell (QML)
- Installer: full-disk encryption default; Secure Boot off; ISO < 6 GB
- Preinstalled: Neovim, Chromium, Obsidian, LibreOffice, plus curated commercial apps (Spotify, Typory, Zoom)

## Three integration layers (low → high trust)

1. **AUR** — anyone publishes a PKGBUILD. Used casually. No curation, no review.
2. **OPR** — `omacom/omarchy-pkgs`. Curated first-party repo. Contribution = upstream PR.
3. **Omarchy Plugin Marketplace** — `omacom/omarchy-plugin-marketplace` (288⭐, 1.1k issues, 46 forks; Sept 2026). ~3,548 community plugins, 16 semantic communities, ~10,733 links. Submission via GitHub issue form. **Automated Security Baseline** scans for `curl|sh`, unpinned remote exec, dangerous sudoers.

No fees, no payments, no per-install billing anywhere in the Omarchy stack.

## Existing crypto plugins (Sept 2026)

| Plugin | Category | What it does |
|---|---|---|
| Bitcoin bar (#3206) | Widgets | mempool.space + coingecko price ticker |
| HODL — Bitcoin Tracker (#4073) | Widgets | Portfolio view |
| Crypto price check (#4010) | Widgets | BTC/ETH/SOL SVG icons |
| Coinbase (#7520, #4997) | Widgets | Read-only via Cloudflare OAuth broker |
| Omacoins (#4811) | Widgets | Watchlist |
| Markets (#4546) | Widgets | Aggregated prices |
| omarchy-converter (#6841) | Productivity | Currencies + crypto + units |

**No BCH plugin. No Electron Cash packaging. No Cashtab, Paytaca, or Cashonize.** All existing crypto plugins are read-only — none can sign and broadcast a transaction.

**OmaPicks** (weekly community ranking) lists 41 categories. None for "Crypto" or "Wallet" — adjacent categories are Network & VPN, System, Security & Privacy.

## Why this matters for bch-bot

1. **No BCH competition.** The only existing BCH desktop wallet on Arch is Electron Cash (AUR, not Omarchy-curated). Omarchy users have no first-party BCH option.
2. **No "spendable" crypto plugin.** Coinbase's plugin is read-only by API design. A BCH wallet plugin would be the **first spendable crypto plugin in the marketplace**.
3. **Omarchy users are keyboard-first power users** — exactly the kind of user who would *use* a BCH wallet hotkey (`Super+Alt+Space → bch send <addr> <amount>`).
4. **No per-tx monetization precedent** anywhere on Linux. The opportunity is green-field.
5. **The Coinbase plugin's existence is the counter-positioning** — a self-custodial, no-KYC, no-OAuth, fully sovereign BCH wallet is philosophically the opposite. That's marketing gold, not a friction point.

See `references/omarchy-research.md` for the full research note (~2,260 words) and `syntheses/bch-bot-omarchy-monetization-plan.md` for the integration + monetization strategy.

## See also

- [[references/x402-on-bch]] — assessing HTTP 402 micropayments against the Omarchy
  wallet: why BCH is absent from the protocol, and what a BCH client would even pay
