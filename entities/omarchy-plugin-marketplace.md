---
pageType: entity
id: entity.omarchy-plugin-marketplace
description: The Omarchy Plugin Marketplace at plugins.omarchy.org — manifest schema (schemaVersion: 1), submission workflow, security baseline, and the catalog of 3,548+ community plugins.
sourceUrl: https://github.com/omacom/omarchy-plugin-marketplace/blob/main/README.md
---

# Omarchy Plugin Marketplace

> Verified against the live docs at `plugins.omarchy.org` on 2026-09-19. Companion to `entity.omarchy.md` (distro overview) and `syntheses/bch-bot-omarchy-monetization-plan.md` (the bch-bot distribution strategy).

## What it is

The Omarchy Plugin Marketplace is the public catalog of community-built QML plugins for Omarchy 4.0 ("Quattro"). Catalog lives at [plugins.omarchy.org](https://plugins.omarchy.org). Source repo at `github.com/omacom/omarchy-plugin-marketplace`. **3,548+ plugins, 16 semantic communities, ~10,733 links** (per the subagent research). Hosted by the Omacom Foundation (nonprofit, ~$12M+ in donations).

Plugins are pure QML + JavaScript. They live in a public GitHub repo with a `manifest.json` at the root. **No backend, no install scripts, no package registry.** The Omarchy shell clones the repo into the user's plugins dir.

## Manifest schema (schemaVersion 1)

Source: an actual first-party plugin manifest at `github.com/omacom/omarchy/tree/main/shell/plugins/agents/manifest.json`.

```json
{
  "schemaVersion": 1,
  "id": "io.github.yourname.plugin-name",
  "name": "Plugin Name",
  "version": "1.0.0",
  "author": "Your Name",
  "license": "MIT",
  "description": "What the plugin does.",
  "kinds": ["bar-widget"],
  "activation": "on-demand",
  "entryPoints": {
    "barWidget": "Panel.qml"
  },
  "barWidget": {
    "displayName": "Bar Widget Name",
    "description": "What shows up in the bar widget list.",
    "category": "Productivity",
    "aliases": ["bch", "wallet"],
    "allowMultiple": false,
    "defaults": { /* user-configurable defaults */ },
    "schema": [ /* settings UI schema */ ]
  }
}
```

### Required fields

- **`schemaVersion`** — `1` (only one supported today)
- **`id`** — globally unique, lowercase, hyphenated. **Reserved namespace: `omarchy.*` (first-party).** Recommended community shape: `io.github.<yourname>.<plugin>`. **Permanent** — IDs from retired listings stay reserved.
- **`name`** — human-readable
- **`version`** — semver
- **`author`** — your name or org
- **`license`** — SPDX identifier (MIT, Apache-2.0, etc.)
- **`description`** — one-line summary
- **`kinds`** — array of plugin types (`bar-widget`, `panel`, `command`, `lock-screen`, etc.)
- **`entryPoints`** — maps kind → entry-point file (relative path inside repo)

### Optional fields

- **`activation`** — `"on-demand"` (default) | `"always"` | `"startup"`
- **`<kind>`** — kind-specific config block (the `barWidget` object above)
- **`preview.png`** at repo root — 50 MB / 40 MP limit, marketplace auto-crops to card/detail images

## Submission workflow

Source: `github.com/omacom/omarchy-plugin-marketplace/blob/main/SUBMISSION.md`.

1. **Public GitHub repo** containing:
   - `manifest.json` at root
   - Root `README.md` with installation + removal instructions
   - Root license file documenting external dependencies
   - Optional root `preview.{png,jpg,jpeg,webp,avif}`
2. **Open a GitHub issue** at `omacom/omarchy-plugin-marketplace` using the [submission template](https://github.com/omacom/omarchy-plugin-marketplace/issues/new?template=submit-plugin.yml). Issue body:
   ```
   ### Repository URL
   https://github.com/yourname/your-plugin
   
   ### Category
   selected_category
   
   ### Tags
   selected_tag, another_selected_tag
   
   ### Suggest a missing tag
   _No response_
   
   ### Maintainer notes
   _No response_
   
   ### Submission checklist
   - [x] The repository is public and contains installation and removal instructions.
   - [x] I have documented the plugin license and any external dependencies.
   - [x] I confirm that I own or have permission to submit this plugin and its preview assets.
   - [x] The plugin does not overwrite user configuration without explicit consent.
   - [x] I understand that approval is for listing and is not a security review.
   ```
3. **Automated validation** runs (checks schema, structure, file existence).
4. **Automated Security Baseline** runs (see below).
5. **Maintainer decision** — only `approved-and-verified` label publishes.

### Categories (9, exact spelling)

`Appearance`, `Desktop`, `Developer Tools`, `Hardware`, `Kids`, `Productivity`, `System`, `Widgets`, `Other`

### Tags (1-3, exact spelling)

`ai`, `bar`, `education`, `games`, `hyprland`, `kids`, `launcher`, `media`, `power-management`, `quickshell`, `security`, `system`, `vpn`, `workspaces`

### Verification

After listing, updates go through the [plugin verification form](https://github.com/omacom/omarchy-plugin-marketplace/issues/new?template=verify-plugin.yml). Either verify the current listed snapshot (exact-SHA required) or publish a newer commit. New listings must pass `approved-and-verified` for publication.

## Automated Security Baseline

Source: `github.com/omacom/omarchy-plugin-marketplace/blob/main/SECURITY.md`.

The baseline is **deterministic, snapshot-based, and intentionally narrow**. It does not run plugin code. It reads selected files from the exact commit SHA produced by submission/validation. It identifies specific static patterns, not all unsafe behavior.

### What it catches (review-required or fail-closed)

| Pattern | Description |
|---|---|
| `curl-pipe-shell` | `curl`/`wget` piped to a shell, or written to a file executed without verification |
| `cargo-git-unpinned` | `cargo install --git` without a full 40-character `--rev` |
| `remote-git-execution-unpinned` | Code from an external git repo executed without a pinned commit |
| `passwordless-sudoers` | Dangerous `NOPASSWD` sudoers policies |
| `bundled-executable-binary` | ELF/PE/Mach-O binary in plugin tree (capability, requires review) |
| `dangerous-tmp-shared-state` | Privileged process control via predictable shared `/tmp` paths |

### What it does NOT do

- No general data-flow analysis
- No execution of plugin code
- No behavioral testing
- No detection of sophisticated supply-chain attacks

### Approximate outcome values

- `passed` — auto-Verified on existing listings
- `review-required` — selectively blocking findings; maintainer must accept
- `needs-fixes` — selectively blocking findings; fix in a new commit
- Failure modes — fail-closed (cannot be approved without complete result)

### Scan limits

- 1,000 relevant files
- 8 MiB total relevant text
- 512 KiB per file
- 50 MB / 40 MP preview
- 256 setup-named candidates max

## Engagement metrics (anonymous)

The marketplace shows:
- **Detail views** (anonymous aggregate)
- **Successful command copies**
- **Hearts**

**NOT shown:** download counts, installation counts, unique users, verified votes, rankings, security signals. **No accounts, no cookies, no IP storage.** Cloudflare holds request metadata at the edge for rate-limiting only.

## Engagement / monetization posture

- No fees, no payments, no per-install billing anywhere in the Omarchy stack.
- The marketplace does not take a cut of plugin-side revenue.
- A per-tx fee **inside the plugin's wallet/business logic** is invisible to Omarchy Core.
- See `syntheses/bch-bot-omarchy-monetization-plan.md` for the bch-bot monetization model built on this principle.

## What's already in the catalog (Sept 2026)

From the earlier subagent research + this verification pass:

- **6 crypto plugins** (price tickers, Coinbase read-only portfolio) — none are BCH, none are spendable
- **41 OmaPicks categories** (weekly ranking) — none for "Crypto" or "Wallet"
- **3,548+ total plugins** (per the catalog stats)
- **First plugin competition** (Aug 2026): Radio Atlas, Omagotchi, AirPods

## BCH gap

- No BCH plugin
- No Electron Cash packaging
- No Cashtab, Cashonize, Paytaca
- No spendable crypto plugin of any kind

**The bch-bot plugin would be the first spendable crypto plugin in the marketplace.** See `syntheses/bch-bot-omarchy-monetization-plan.md` for the plan.

## References

- [Omarchy Plugin Marketplace](https://plugins.omarchy.org)
- [Submit a Plugin](https://plugins.omarchy.org/publish.html)
- [Marketplace README](https://github.com/omacom/omarchy-plugin-marketplace/blob/main/README.md)
- [SUBMISSION.md](https://github.com/omacom/omarchy-plugin-marketplace/blob/main/SUBMISSION.md)
- [SECURITY.md](https://github.com/omacom/omarchy-plugin-marketplace/blob/main/SECURITY.md)
- First-party reference: `github.com/omacom/omarchy/tree/main/shell/plugins/agents/manifest.json`
- Companion entity: `entity.omarchy.md`
- Companion plan: `syntheses/bch-bot-omarchy-monetization-plan.md`
