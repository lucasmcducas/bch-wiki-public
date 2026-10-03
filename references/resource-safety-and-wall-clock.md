---
pageType: reference
id: reference.resource-safety-and-wall-clock
description: Why a test runner bricked a 4-core machine, how a leaked wallet counter hid real money, and the two rules that prevent both — self-excluding discovery and state that only moves on success. Written from a live incident on 2026-10-02.
sourceUrl: internal/reference
---

# Resource safety and the wallet's own clock

Two failures from the same session, on the same 4-core / 8 GB Omarchy box, that
share one root cause: **code that discovers other code, and state that moves
whether or not the work succeeded.** Both are easy to get right and I got both
wrong while building test infrastructure for a wallet holding real funds.

This page is the operational counterpart to
[`failure-modes-we-hit.md`](../syntheses/failure-modes-we-hit.md), which catalogues
transaction-construction bugs. Those bugs lose or misdirect money. These two
never produce a wrong transaction — they make the machine unusable and the
ledger unreadable, which is arguably worse because nothing announces itself.

---

## 1. A test runner that discovered itself and forked

`scripts/test-all.mjs` was written to fix a real gap: `send` and `round-trip`
were dead (they called `deriveReceivingAddresses` without importing it) and 433
assertions passed anyway. The runner's second phase executed every entry point,
which is the check that catches a missing import.

It found test files like this:

```js
const tests = readdirSync(scriptsDir).filter((f) => f.startsWith('test-'));
```

**That glob matches the runner itself.** It is a `.mjs` file in `scripts/` whose
name starts with `test-`. So it spawned a child that ran the same discovery and
found the same 21 files, one of which was the child. Each generation spawned the
next. Node's `spawnSync` waits for a child, the child enqueues another, and
nothing ever terminates.

Observed on omarchy, 2026-10-02:

| Metric | Value |
|---|---|
| `node` processes | **835**, all `test-all.mjs`, chained parent → child |
| Load average | **22.35** on 4 cores |
| Free memory | **99 MB** of 7.4 GB |
| Result | SSH unresponsive; box required a reboot |
| Recovery | `pkill -9 -f test-all.mjs` → 3 processes, 5.3 GB free, load falling |

Two runs overlapped because I started a second without checking whether the first
was still going, which doubled the fork rate. Nothing bounded it: no
self-exclusion, no concurrency ceiling, no memory cap, no `nice`/`ionice`.

**The fix was deletion, not repair.** A plain shell loop over the test files costs
nothing and covers the same ground. The entry-point smoke test is worth keeping —
in a form that runs one script at a time and never enumerates itself.

### Rules

- **A script that discovers other scripts must exclude itself, by resolved path
  — not by name pattern.** `import.meta.url` compared against each candidate is
  exact; a substring test is a guess. This is the only form I would trust.
- **Never let a checker execute what it discovered without a bound.** If a
  discovery step is dynamic, the execution step needs an explicit cap and a
  per-item timeout.
- **Check what is already running before launching anything.** Twice in this
  session I started a second long-running job while the first was live. `pgrep`
  first, always.
- **Run heavy work on a remote box under `nice`/`ionice` with a memory cap**, and
  check free memory and load before starting.
- **A resource limit is a correctness property, not a politeness setting.** Code
  that can wedge the machine it runs on is itself a bug, and the failure mode is
  indistinguishable from hardware failure.

### The diagnostic that would have caught it in seconds

```sh
pgrep -c node          # 835
ps -eo pid,ppid,pcpu,args --sort=-pcpu | head   # 1000% node, PPID chain
```

I asked "am I using too many resources?" and answered with a *theory* about
WebSocket leaks while a fork bomb was saturating every core. **When someone
reports the machine is struggling, run `pgrep` and `uptime` before theorising.**
A guess costs credibility and time; a measurement costs a second.

---

## 2. A leaked counter hid 855,311 sat

`newChangeAddress()` derived an address and **persisted the increment on
call**. So any run that reached signing consumed a change index — including runs
the network then rejected.

Measured, 2026-10-02:

- A swap signed correctly and was rejected with `Missing inputs` (Cauldron pool
  contention — a competing swap consumed the pools between build and broadcast).
- That single run advanced `change_index` by **5**.
- Five such runs pushed it **40 → 45**.

The counter was not the visible problem. The visible problem was this:

```js
const CHANGE_GAP = 20;   // balance.mjs, utxos.mjs, round-trip.mjs all shared this
```

`balance` scanned change indices 0–19. The wallet's BCH sat at **index 38**.

```
before:  0.00803 BCH,  4 UTXOs,  ROACH 1.1
after:   0.01659311 BCH, 6 UTXOs, ROACH 2
```

**The money was never lost. The tool could not see it.** The leak inflated the
counter, and the fixed gap limit could not reach what the counter pointed at. Two
bugs, one causal chain.

The fix was to derive the scan window from the wallet's own state, with a floor
and a ceiling:

```js
const SCAN_FLOOR = 20, SCAN_CEILING = 200, FORWARD_WINDOW = 5;
const scanCount = (key, state) =>
  Math.min(SCAN_CEILING, Math.max(SCAN_FLOOR, Number(state[key] ?? 0) + FORWARD_WINDOW));
```

The ceiling matters independently: a corrupted or inflated counter must not be
able to trigger thousands of derivations on a small box.

### Rules

- **State that records progress must only advance on success.** `newChangeAddress`
  now takes `commit = true|false`; scripts derive without committing and call
  `commitChangeAddress(index)` after the node returns a real 64-hex txid. The
  commit is idempotent and takes the highest index, so several deferred addresses
  are covered.
- **Key the guard off the outcome, not the intent.** `swap.mjs` originally guarded
  on `BCH_CONFIRM === 'yes'` — present intent. That still leaked, because a run
  *with* intent can fail. Only acceptance is a safe trigger.
- **Deriving an address is free; reserving one is the part that costs.** Unused
  addresses are harmless, so the failure mode is waste rather than loss — but a
  wallet that mutates its own state on a failed transaction is a wallet whose
  state you cannot reason about.
- **Never scan a fixed window.** A UTXO can exist at any index the wallet has ever
  derived. The counters already record the highest; use them.
- **A gap limit is a safety bound, not a reporting window.** Deriving is cheap;
  querying a node for 200 addresses is not free, which is why the ceiling exists.

---

## Why these two belong together

Both are failures of **bookkeeping about work**, not of the work itself. The
transactions were correctly constructed, correctly signed, and correctly
rejected. The wallet's *own record* of what it had done was wrong in both cases —
once in process tables, once in `state.json`.

A wallet whose bookkeeping is unreliable cannot be audited after the fact, cannot
be reasoned about, and cannot be trusted to tell you where your money is. That is
the same class of defect as a wrong signature, and it deserves the same rigour.
