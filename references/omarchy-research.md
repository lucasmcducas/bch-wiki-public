---
pageType: reference
id: reference.omarchy-research
description: Research note on Omarchy Linux distribution — what it is, how third-party integrations work, and where a self-custodial BCH wallet with per-transaction monetization could plug in.
sourceUrl: https://en.wikipedia.org/wiki/Omarchy
---

# Omarchy Research Note

## 1. What is Omarchy?

Omarchy is an open-source Linux distribution created by **David Heinemeier Hansson (DHH)** — the creator of Ruby on Rails and co-owner of 37signals — with **Ryan Hughes** as co-developer. It was first released on **June 26, 2025**, initially as an opinionated post-install configuration of Arch Linux + Hyprland aimed at developers, and grew into a full distribution with its own installation ISO and its own package repository. The current major release is **4.0 ("Quattro")**, released **August 14, 2026**, which replaced several Wayland components (Waybar, Walker, Mako, SwayOSD, hyprlock, polkit-gnome) with a unified desktop shell built on Quickshell.

In August 2026 Hansson incorporated the **Omacom Foundation** (nonprofit) to fund continued development; early patrons include 1Password and 37signals ($300K each), plus individual commitments totalling $12M from figures including Tobias Lütke (Shopify), Patrick Collison (Stripe), Michael Dell, Jack Dorsey (Block), Brian Armstrong (Coinbase), Drew Houston (Dropbox), Peter Steinberger (OpenClaw), and others. DigitalOcean added a $3M commitment in September 2026. **Coinbase CEO Brian Armstrong is an explicit donor.** Source: Wikipedia, *The Register*, *ZDNet*, Omarchy News. License: MIT.

## 2. Architecture

- **Base:** Arch Linux (rolling). Uses `pacman`, the standard Arch package manager.
- **Package repository:** Omarchy maintains its own **Omarchy Package Repository (OPR)** — multi-channel (edge → rc → stable), multi-arch (x86_64 + aarch64), built from in-tree PKGBUILDs via `omacom/omarchy-pkgs`. Channels are advanced forward-only; the release train is operated by `bin/omarchy-release`. Signing key fingerprint is documented in the repo.
- **Kernel:** Bespoke `linux-omarchy` kernel, installed as default boot option (release notes for 4.0.x mention this).
- **Init/systemd:** Standard Arch systemd.
- **Display server:** Wayland, exclusively.
- **Compositor:** Hyprland (tiling).
- **Desktop shell:** **Quickshell** (QML-based, replaces the previous collection of Wayland utilities in v4). Status bar, launcher, notifications, menus all live inside Quickshell.
- **Preinstalled software:** Neovim, Chromium, Obsidian, LibreOffice, Kdenlive, OBS Studio, plus dev tools, AI coding agents, terminal apps, and a curated selection of commercial software (Spotify, Typora, Zoom, Alacritty).
- **Installer:** Full-disk encryption by default; ISO under 6 GB; sub-minute installs reported. The user must disable Secure Boot / TPM (Omarchy explicitly calls these "Microsoft security schemes").

## 3. Wallet / crypto integration patterns on Linux distros

Three distinct UX patterns exist today:

- **Tails (amnesic):** Ships **Electrum** as a built-in app, integrated into the Tails "Persistent Storage" feature so the wallet survives reboots. Persistence uses an encrypted partition; the wallet file is stored there. Documented in `tails.net/doc/anonymous_internet/electrum`. The integration is first-party — no payment, no marketplace. (BTC only; no BCH.)
- **Ubuntu / Debian / Fedora:** No first-party wallet. Users install via `apt` / `dnf` / Flatpak / Snap, or grab AppImages. No distro-level UX for crypto.
- **Arch / Manjaro:** Bitcoin Core lives in `[community]`; Sparrow, Electrum, Electron Cash, Bitcoin Cash Node, Bitcoin ABC, Cardano wallet, etc. all live in the **AUR** (user-submitted, anyone can publish). Pattern is "everyone maintains their own PKGBUILD"; the distro does not curate or surface them.

In all three cases, **per-transaction monetization by the distro is absent.** Tails is donation-funded; Arch doesn't even have a paid tier; Ubuntu and Fedora take vendor sponsorship (Snap/Flatpak deals) but not per-app fees.

## 4. Existing crypto wallet integrations on Arch / Arch-based

AUR packages I verified:

- `bitcoin-bin`, `bitcoin-qt-bin`, `bitcoin-cli-bin` — Bitcoin Core binaries
- `bitcoin-git` — build-from-source variant
- `sparrow-wallet`, `sparrow-wallet-git`, `sparrow-wallet-reproducible` — Sparrow (BTC)
- `electron-cash`, `electron-cash-bin`, `electron-cash-git`, `electron-cash-slp` — **Electron Cash (BCH) and its SLP fork** (maintained in AUR; uses `python-pathvalidate`, has had Python compatibility issues recently)
- `cardano-wallet-bin`
- `bitcoin-abc` (referenced in r/btc)

Electron Cash is the canonical existing BCH desktop wallet for Arch. There is **no Cashtab, no Paytaca, no Cashonize, no BCH-specific AUR package beyond Electron Cash.** This is a clear gap.

## 5. Omarchy's package submission workflow

**Three layers, in increasing order of trust:**

1. **AUR** — anyone can upload a PKGBUILD. Documented in the Omarchy manual: *"just use Install > AUR. Just remember that the AUR isn't vetted by the Arch team. It's like RubyGems or npm. Anyone can upload."* Used casually; not curated.

2. **Omarchy Package Repository (OPR)** — `omacom/omarchy-pkgs`. PKGBUILDs live under `pkgbuilds/<package>/` with `.omarchy/package.json` metadata (upstream watch / build hooks). Releases are signed and advanced through channels. This is the curated first-party repo. **Contribution path is upstream PR to `omarchy-pkgs`** — no public form, no fees.

3. **Omarchy Plugin Marketplace** — `omacom/omarchy-plugin-marketplace` (288 stars, 1.1k open issues, 46 forks as of Sept 2026). This is the public-facing distribution surface. **3,548 community plugins, 16 semantic communities, ~10,733 links.** Submission workflow (per `SUBMISSION.md`):
   - Public GitHub repo containing `manifest.json`, root README, license file, optional `preview.png`
   - One of nine categories: **Appearance, Desktop, Developer Tools, Hardware, Kids, Productivity, System, Widgets, Other**
   - 1–3 tags from a fixed list (ai, bar, education, games, hyprland, kids, launcher, media, power-management, quickshell, security, system, workspaces) — or suggest a new reusable tag
   - Globally unique plugin ID, namespaced (`io.github.yourname.plugin-name`); `omarchy.*` is reserved
   - Submit via GitHub issue on `omacom/omarchy-plugin-marketplace` (form or scripted body), then automated validation runs, then **Automated Security Baseline** runs
   - Baseline detects deterministic bad patterns: download-to-shell execution, unpinned remote git source exec, dangerous passwordless sudoers, privileged process control via shared temp state
   - Publish requires **explicit approved-and-verified maintainer decision** after the baseline passes
   - Engagement metrics: anonymous aggregate views, command copies, "hearts"

   **No fees, no payments, no per-install billing infrastructure.** The marketplace runs as a community service under the Omacom Foundation.

## 6. Recent (2025–2026) Omarchy + crypto news

- **Crypto-themed plugins are landing.** As of Sept 2026, the marketplace hosts **at least six** crypto/Bitcoin plugins: *Bitcoin bar* (#3206, mempool.space + coingecko price widget), *HODL — Bitcoin Tracker* (#4073), *Crypto price check* (#4010, Bitcoin/Ethereum/Solana SVG icons), *Coinbase* read-only portfolio widget (#7520, #4997 — uses Cloudflare OAuth broker, wallet:user:read only), *Omacoins* crypto watchlist (#4811), *Markets* by costafot (#4546), *omarchy-converter* (currencies + crypto + units, #6841, #6297).
- **No BCH plugin exists.** No Electron Cash packaging, no Cashtab, no Cashonize, no Paytaca.
- **No "Crypto / Wallet" category.** OmaPicks (the weekly community ranking, `omapicks.com`) lists 41 categories — none are crypto or wallet. Adjacent: Network & VPN, System, Security & Privacy, Productivity. A new plugin can suggest a category, but the marketplace taxonomy does not yet acknowledge wallets as a class.
- **Omarchy 3.7 (May 2026)** unified the CLI and expanded gaming support.
- **Omarchy 4.0 Quattro (August 2026)** is the current stable; sub-minute installs, < 6 GB ISO, full Quickshell rewrite.
- **Plugin competition #1 (August 2026)** — DHH paid out of pocket for prizes ($2.5K / $1K / $500). Winners: Radio Atlas, Omagotchi, AirPods. No crypto plugin placed.
- **Foundation established (August 2026)** — formalised the funding structure; expects recurring plugin competitions.
- **Docker-group root vuln (Aug 2026, fixed in 4.0.1)** — added user to `docker` group by default; any desktop app could escalate. Fixed with explicit opt-in + warning. Relevant: the security baseline already checks for dangerous sudo patterns; this matters for any wallet plugin touching privileged services.

## 7. How do other distros monetize optional integrations?

- **Snap Store (Canonical):** Vendor pays nothing to publish free snaps; proprietary snaps go through a manual review and revenue share with Canonical. App revenue share exists for paid snaps (~20% in some accounts). Closed, one-vendor-controlled.
- **Flathub:** Community-run nonprofit (hosted by GNOME Foundation). Discussed paid-app support since 2023 but as of 2026 still primarily free. Stripe-style payment integration was prototyped but not generally available.
- **AUR (Arch):** No monetization. Maintainers are volunteers; some link to donation pages.
- **nixpkgs:** No monetization.
- **Mac App Store, Windows Store, iOS App Store:** 15–30% cut; standard reference point for per-transaction models.
- **Chrome Web Store / Firefox AMO:** Extensions are free; the host doesn't take a per-use cut; monetization is via the developer's own site or via the extension itself collecting payment.
- **BCH precedent:** BCH ecosystem runs on direct merchant adoption, payment processors (BitPay, CashPay, Merchant API), and self-hosted POS (BCH Merchant PoS by SoftwareVerde, Cashonize web wallet). **No BCH wallet currently runs a per-transaction "toll" through a Linux distro channel.**

The closest precedent for a Linux-distro-mediated per-transaction revenue model is the Snap Store — but Canonical takes the cut, not the developer, and there's no analogue on Arch / Omarchy.

## 8. Constraints to know

- **Target audience:** Developers, keyboard-first power users, "modern savvy computer user" (per the manual). Not aimed at end-users / parents / enterprise.
- **Not privacy-focused in the Tails sense:** Omarchy is opinionated about *aesthetics* and *DX*, not anonymity. The installer requires Secure Boot off and uses full-disk encryption for the install drive, but it's not an amnesic distro.
- **Not commercial in the proprietary sense:** MIT-licensed core, Omacom Foundation is nonprofit, marketplace is community-curated. The Foundation is funded by donations, not by fees.
- **Political controversy:** Wikipedia documents backlash over Hansson's political statements and the 1Password sponsorship; some commentators (Matthew Garrett) have argued the project is "incompatible with the goals of free software." **Any Omarchy integration will inherit that social context** — independent of whether BCH itself is politically neutral.
- **Wayland / Quickshell-only:** Plugins are QML + Quickshell, or scripts. Any wallet plugin should fit the Quickshell widget model (bar widget + IPC command) for the best UX.
- **Keyboard-first:** Hotkeys (`Super + Space`, `Super + Alt + Space`) are how users reach features; visual desktop-only integration is lower value.
- **Security baseline is strict:** The marketplace explicitly scans for `git clone https://... | sh` patterns, unpinned upstream exec, and dangerous sudoers. A wallet plugin that pulls `npm install` or `pip install` of an un-pinned tarball will trigger `security-review-required`.
- **Brian Armstrong (Coinbase CEO) is a named Omacom donor.** Coinbase has a *read-only* plugin in the marketplace. A BCH plugin would be a deliberate philosophical counterpoint to that — and given the FOSS-fork history of BCH vs BTC, that tension may be commercially advantageous *or* a friction point.

## 9. "Omarchy" naming — confirmed

The naming is **"omakase" (お任せ)**: "leave it to the chef" / curated pre-picked set. DHH explicitly frames it as: *"It's not just a grab bag of preinstalled packages… It's a complete system designed with both aesthetics and productivity in mind. Because a beautiful system is a motivating system."* The manual header reads: *"Omarchy is an omakase Linux distribution based on Arch."*

This is the same naming lineage as **Omakub** — DHH's earlier Ubuntu-based developer setup (still referenced as a predecessor). Omarchy is Omakub's successor, re-platformed on Arch + Hyprland and now grown into a distribution. The name pattern (Oma-) and the philosophy (curated, opinionated, "what DHH uses") are direct carry-overs.

## Monetization opportunities for bch-bot on Omarchy

Based on the evidence above, here are the concrete integration paths and how each one can monetize per-transaction:

1. **Omarchy Plugin Marketplace entry — "BCH Wallet" widget + command.** Category: *Widgets* or *Productivity* (suggest *Wallets* as a new category if a single crypto-wallet plugin is too small to justify it alone). Manifest a QML bar widget for balance + recent activity, an IPC command (`omarchy bch send <addr> <amount>`) for one-shot payments, and a full-screen modal (similar to Omasticky for notes) for the seed-phrase flow. **Monetization:** plugin is free; revenue model is a per-tx fee inside the wallet itself — the user pays BCH to broadcast, or a tiny BCH service fee goes to a treasury address configured in the plugin. The marketplace doesn't take a cut; the per-tx fee is invisible to Omarchy Core.

2. **Omarchy Package Repository (OPR) — Electron Cash or a custom Node wallet.** Add a `pkgbuilds/bch-bot/` package. The OPR's automated build pipeline signs and ships it via stable channel. **Monetization:** the package ships with a default treasury address (or a config flag for it); per-tx fees flow in BCH at the libauth layer. The OPR doesn't gate this.

3. **AUR-only — fast lane.** Publish a `bch-bot` PKGBUILD against the AUR today, with an `omarchy` opt-in post-install hook that adds it to the Omarchy menu (`Install > Package` will find it via pacman once AUR is enabled). No review needed. **Monetization:** same per-tx fee in the libauth transaction builder.

4. **First-party integration pitch.** Because Coinbase is a *read-only* plugin and there's no spend-from-Omarchy plugin, an *Omarchy-native, self-custodial, spendable* BCH wallet is a categorically different offering. It's the kind of project that could plausibly attract DHH attention in a future plugin competition — the competition prizes (last round $4K total, DHH has signalled recurring competitions via Foundation funding) would offset some integration cost.

## Caveats and unknowns

- I could not find a documented fee or revenue-share arrangement inside the Omarchy Plugin Marketplace; based on the README and SUBMISSION.md the marketplace is non-commercial. **No precedent for taking a cut of plugin-side revenue.**
- The Omarchy manual v4 pages for the actual install + package flow were not deeply inspected beyond `omarchy pkg add` references; the v3 manual is the most thoroughly documented.
- I have not verified whether Electron Cash currently builds cleanly on Omarchy 4.0 / Quickshell / Wayland — likely yes (Electron Cash is a Qt app), but unverified.
- The Omacom Foundation's future commercial posture is unknown — donations today, but no public pricing or paid-channel plans surfaced.
- "Per-transaction monetization" inside a BCH wallet on Linux has **no live precedent** I could find; this is a green-field model. The closest analogues are BitPay merchant fees and the historical Bitcoin Core `paytxfee` setting (developer-set, not enforced).
