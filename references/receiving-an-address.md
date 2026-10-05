---
pageType: reference
id: reference.receiving-an-address
description: Turning a CashAddr into something a payer can actually use — the QR as a state-verification target, the clipboard as an integrity boundary, and the three bugs that each passed as working. Written from fixes shipped in bch-bot-omarchy, 2026-10-04.
sourceUrl: internal/reference
---

# Receiving an address

A wallet that cannot be *paid* has no interesting security properties. Every
protection in [[security/wallet-threat-model]] — hardware signing, single-use
addresses, the clipboard-hijacker entry — is downstream of one thing: the payer
must transfer funds to the address you showed them. Three transports carry that
address, and all three fail quietly:

| transport | how it fails | what it looks like |
|---|---|---|
| **the text** | user retypes it, or pastes the wrong thing | transaction goes nowhere |
| **the QR** | present but not decodable, or encoding the wrong string | phone camera shows a square, payment fails |
| **the clipboard** | empty, or stale from a previous copy | user pastes a valid-looking but *wrong* address |

The dangerous one is the last. An empty clipboard is obvious; a clipboard holding
a **previous** address is indistinguishable, at a glance, from the right one.

This page is the receiving-side counterpart to
[[references/driving-omarchy]] — the first is about getting the address out of
the wallet, this is about getting it into someone's payment app correctly.

## Why a fresh address per request is the right default

`bch-bot address` derives a **new** address on every call, walking the HD index.
This is not a cache-miss bug; it is the property that makes receive unlinkable.
Each receive lands on its own UTXO, so two payments to "the wallet" cannot be
joined by inspecting the output set.

The cost is real and worth stating plainly: **there is no stable address.** A
payer cannot be given one to hold against, and asking for "the address" twice
returns two different values. This shows up in three places:

- a support conversation ("is this the right address?")
- an invoice or donation page that caches a payout address
- anyone comparing the panel against a sender's expectation

Fresh-per-request is still the correct default for a desktop wallet. Address
*reuse* is what breaks unlinkability, and a receiving cache would be a
regression in the property that matters. If a stable address is ever needed, the
right shape is a **rotating set** disclosed as such — not a single cached
address, and never one silently reused across receives.

## The QR is a verification surface, not decoration

A QR's job is to get the address into a phone camera with **zero** transcription
error. That makes it the strongest of the three transports, and it is worth
knowing why it is trusted more than the text next to it:

- The text on screen can be truncated, elided, or wrapped. A user copying from a
  wrapped address gets a fragment.
- A 42-character CashAddr is not checkable by eye. It has a
  [checksum](/security/script-and-signing) — that is what the encoding exists for.
- The QR encodes the string the panel *actually holds*, so it is a
  **cross-check** on the text. Decode it and compare. If they differ, something
  upstream is wrong, and a human comparing the two strings by eye would not
  catch it.

That last property is the one worth building a test around. Encoding a
displayed string and a held string are different code paths, and the failure mode
where they diverge is exactly the case where the QR looks fine and the text does
not.

### Generate it from the address, not alongside it

An encoder invocation belongs **on the address-changing path**, not in a render
path. A BCH address is 42 characters; encoding is cheap but not free, and a
process spawn plus a file write per repaint is waste that a repaint loop pays
every frame.

The generation must be **input-validated and fail-closed**:

- Validate the CashAddr shape (prefix plus exactly 42 chars of the base32
  alphabet) *before* it reaches any command. An unvalidated derived address on a
  command line is a shell-injection surface, and the check doubles as a
  correctness check — a malformed address produces no QR rather than one that
  scans to the wrong thing.
- Adopt the generated file **only on success**. A failed encode leaves the
  previous QR in place otherwise, which is the worst outcome available: a valid
  QR for a *previous* address, still scannable, still wrong.
- **Hide** the image when there is no QR. A present-but-unscannable code is
  worse than no code: it invites a scan attempt that fails at the payment step
  with no hint why.

### Put it on a white plate

A QR rendered on a dark panel background will not scan on most phone cameras.
Dark-theme inverted QRs are not reliably supported by the scanner software in
the wild. Render a light plate behind the code — this is a hard requirement of
the transport, not a styling preference, and it is the kind of detail that gets
dropped by a UI pass that treats the widget as a picture rather than as a
payment surface.

## The clipboard is an integrity boundary

A clipboard is a **shared, global, unvalidated** buffer. Three distinct
failures live there, and they need different fixes:

1. **Empty** — the copy silently did nothing.
2. **Stale** — the copy did nothing, and a *previous* value is still there. The
   user pastes something valid-looking that is not their intent.
3. **Corrupted** — a clipboard hijacker replaced the address between copy and
   paste. This is the entry already in [[security/wallet-threat-model]], and it
   is the reason the address on a hardware screen remains the final check.

**Prefer passing the value as an argument, not down a pipe.** A UI that starts a
clipboard helper and then writes the value to its stdin has a race: the pipe
does not exist until the child opens it, and a write issued before that is
dropped. Every mainstream clipboard tool accepts the value as an argument, and
using that removes the window entirely rather than narrowing it.

If you must stream it, **foreground the child**. A clipboard helper that forks
exits immediately, so a supervisor sees success before the clipboard is actually
owned — a "copy succeeded" state that is reporting on the wrong event.

The verification loop that works, and it is only four commands:

```bash
# 1. clear the clipboard completely
pkill -x wl-copy
wl-paste            # -> "Nothing is copied"

# 2. user clicks Copy

# 3. read it back
wl-paste
```

Step 1 is the part that gets skipped, and skipping it is how a *correct* copy
gets reported as broken: a leftover clipboard owner from an earlier test still
holds the buffer, so the panel's new value is never observed. **`wl-paste`
returning `Nothing is copied` is the only trustworthy baseline.**

Two properties of the tools themselves, both of which look like bugs:

- **`wl-copy` blocks over SSH** until the clipboard is replaced. That is what a
  clipboard owner *is*. Background it, and read the result with `wl-paste`.
- `wl-paste --no-newline` avoids a trailing-newline difference between what was
  copied and what is read back, which otherwise reads as a payload mismatch.

## What a passing test has to assert

Presence is not correctness. A QR file existing proves the encoder ran; it says
nothing about whether the code scans or whether it matches the text beside it.
Round-trip the artefact through a real decoder and compare all three values:

```bash
# decode the generated code
convert qr.svg -resize 400x400 qr.png
zbarimg --quiet --raw qr.png
```

Then assert **all three are byte-identical**: the string the panel displays, the
string the QR decodes to, and the string `wl-paste` returns after a real click
from a cleared clipboard. Equality across all three is the only thing that makes
"the address is copyable and scannable" a fact rather than a claim, and it
catches the divergence cases that any single check misses.

Add the adversarial case, which is the one that matters for a wallet: clear the
clipboard, copy address A, then copy address B, and assert the clipboard holds
**B**. Because the address is fresh per request, A and B are guaranteed
different — a stale-buffer bug cannot hide behind two identical values.

## Three bugs, all of which passed as working

Recorded in full in [[syntheses/failure-modes-we-hit]]; the transferable shape
is what they had in common.

**The clipboard write was dropped.** A deferred write into a not-yet-open pipe,
exactly as described above. The button's own hover changed, so the flow looked
alive; only reading the clipboard back showed nothing had happened. *A control's
own affordance responding is not evidence the control did its job.*

**A guard discarded the request.** A shared-process helper that returned early
when already busy, intended as mutual exclusion. The panel refreshes balances in
the background, so "busy" is the **normal** state when a user clicks — so the
user's request was dropped roughly one click in three, and since nothing retried,
the view sat on a spinner forever with no error. Queue the request instead. *A
spinner with no error is a dropped request until proven otherwise.*

**The QR was never built.** A documented, scoped, still-open work item assumed
present. The address and the copy button worked, the panel looked complete, and
a transport that does not exist cannot fail loudly. *Check the deliverable
exists; a plan entry is not an implementation.*

## See also

- [[references/driving-omarchy]] — getting the panel driven and observed headlessly
- [[security/wallet-threat-model]] — the clipboard-hijacker entry and friends
- [[security/script-and-signing]] — the CashAddr checksum the QR protects
- [[concepts/cash-tokens]] — what a receive address carries besides BCH
- [[references/bch-zero-conf-security]] — the other half of receiving safely
