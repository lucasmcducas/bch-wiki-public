# Driving the omarchy box

What it takes to see, click, and verify a Quickshell UI on the omarchy machine —
and the things that had to be discovered the hard way before any of it worked.

This is a record of environment facts, most of which are not discoverable from
documentation and all of which cost time to find. Two of them were first written
down **wrong**, and the corrections are kept in place rather than edited out,
because both wrong conclusions were plausible and cost a round of verification
that observed nothing.

## The machine

| | |
|---|---|
| Compositor | Hyprland 0.56.2 on Wayland |
| Shell | Quickshell 0.3.1, launched as `quickshell -n -p /usr/share/omarchy/shell` |
| Live shell config | `/usr/share/omarchy/shell` — this is where `qs.Ui` and `qs.Commons` live |
| Bar | a **28px vertical strip at x=0**, full height |
| Monitor | 1920×1080@60, name `Unknown-1` |
| GPU | `simple-framebuffer` |

## 1. `qs.Ui` is NOT missing — you are looking in the wrong tree

The panel does `import qs.Ui` for `Style`, and a search of `/etc/xdg`,
`/usr/share/quickshell` and `~/.config/quickshell` returns **nothing**. That
search looks right and is wrong:

```
/usr/share/omarchy/shell/
  Commons/  Ui/  plugins/  services/  shell.qml
```

The real modules are in omarchy's own shell directory, which none of the obvious
paths mention. Installing a wallet plugin means dropping it in
`~/.config/omarchy/plugins/<id>/`, and it gets picked up because the shell
scans that directory — **not** because you configured anything.

**Consequence:** a plugin is not "deployed" when its code is committed. It is
deployed when the installed clone is updated. See §5.

## 2. Screenshots work. `grim` is the wrong tool, and I published the wrong claim

**This section previously said screenshots were impossible on this machine. That
was wrong, it was published, and it cost a full round of bogus verification.** The
error is worth keeping because the same mistake is easy to repeat.

What actually happened: `grim` hangs. Bare `grim`, `-o` with every plausible
output name, `systemd-run --user`, `uwsm app`, signature set and unset — all hang
or fail, and the pattern looks conclusive:

```
WAYLAND_DISPLAY unset     →  grim: "failed to create display"   (instant)
WAYLAND_DISPLAY=wayland-1 →  grim connects, then blocks         (timeout)
```

That reads as "the compositor accepts the client but capture is impossible".
I stopped there and wrote the conclusion down. The DRM connector really is
`simple-framebuffer`, so the diagnosis of *grim's* failure was accurate — and
irrelevant. **One tool hanging is a fact about the tool, not a fact about the
machine.**

The distro ships its own capture, which goes through the compositor rather than
the DRM device:

```bash
export HYPRLAND_INSTANCE_SIGNATURE=<sig> XDG_RUNTIME_DIR=/run/user/1000 WAYLAND_DISPLAY=wayland-1
omarchy capture screenshot fullscreen save
ls -t ~/Pictures/*.png | head -1     # screenshot-2026-10-04_12-28-42.png
```

**`save` is the whole trick.** Omit it and the command opens a 10-second preview
and blocks, the ssh call times out, and you have "proved" capture is broken a
second time. Modes are `screenshot region|windows|fullscreen|scroll`, each
accepting an optional `copy` or `save`.

The only mode needing a human is the interactive region picker, and you do not
need it: capture `fullscreen` from a script and crop locally. Print Screen opens
the picker; Alt+Print Screen records; Super+Print Screen picks a colour;
Super+Ctrl+Print Screen runs OCR on a region.

### Two false readings along the way

Both produced confident nonsense from a single command, and both are cheap to
avoid:

```
$ dd if=/dev/fb0 bs=4 count=8 2>/dev/null | od -A x -t x1
000000
```

That looks like a blank framebuffer. It is **permission denied** — `/dev/fb0` is
`root:video 0660`, and luke is not in `video`. I had suppressed stderr in the
very command whose purpose was to find out why it failed.

The other: the wallpaper is **animated**. A detector that said "this region
changed, so a panel opened" reported a hit at y=80 that was pure wallpaper
motion. Calibrate any such threshold against several samples taken seconds
apart, or diff two captures taken back to back.

`hyprctl layers` remains the cheap structural check — it is how the 28px bar and
the 1920×1080 output were identified — but it is no longer a substitute for
looking.

## 2b. The screensaver silently invalidates an entire verification pass

`org.omarchy.screensaver` covers the whole screen after `shell.json`'s
`idle.screensaver` (900s). It is a `foot` terminal running `omarchy-screensaver`
and it needs no password to leave.

While it is up, the failure is **silent and looks like a widget bug**. The
signature is a set of three individually reassuring facts:

| observation | what it seems to say | what it actually means |
|---|---|---|
| bar region is 100% `#000000` in the capture | the widget renders nothing | the bar is not in the capture |
| `hyprctl dispatch click R` exits 0 | input was delivered | it went to the saver |
| the widget's `bch-bot balance` is still running | the widget is alive | it refreshes while locked |

So the widget that ran `bch-bot` perfectly while completely invisible is the
crucial case: **a live process is not a visible UI.** The check is one call:

```bash
hyprctl clients 2>/dev/null | grep -c class:   # 0 = desktop visible
pgrep -f org.omarchy.screensaver               # non-empty = locked
```

Wake it with input before every capture and every click:

```bash
hyprctl dispatch movecursor 960 540 && hyprctl dispatch movecursor 961 541
```

A mouse move is enough. If it persists, `kill` the
`foot --app-id=org.omarchy.screensaver` process. It re-arms on the idle timer, so
this is not a permanent change to the machine.

A cheap way to catch a bad capture before blaming the widget: `shell.json` sets
the bar `"transparent": false`, so a real bar paints an **opaque themed
background** (measured `#1A1B26`, plus `#8C93B2` and `#A58F5A` icon colours). Pure
`#000000` across the whole 28px strip means the bar is absent from the capture —
never that the bar is empty.

## 3. `hyprctl` needs the signature from the running compositor

```
$ hyprctl version
HYPRLAND_INSTANCE_SIGNATURE not set! (is hyprland running?)
```

It *is* running. The signature is not in your shell and is not exported to
`uwsm app` either. Read it out of the compositor's own environment:

```bash
pid=$(pgrep -u luke -x Hyprland | head -1)
ls /run/user/1000/hypr/     # the socket dir, named <sig>_<pid>_<ts>
```

Then export it for everything that touches the display:

```bash
export HYPRLAND_INSTANCE_SIGNATURE=<sig>
export XDG_RUNTIME_DIR=/run/user/1000
export WAYLAND_DISPLAY=wayland-1
```

With that set, these all work over ssh:

```bash
hyprctl version
hyprctl cursorpos
hyprctl layers            # the surface tree — your eyes on a headless box
hyprctl dispatch movecursor X Y
hyprctl dispatch click R  # right click
wtype "text"              # type
```

`hyprctl dispatch` is enough to drive a real UI: it moves a real cursor and
sends real clicks to the real compositor.

## 4. Two signals do not fire, and that decides how you test

Measured in this Quickshell build, on a bare `ShellRoot`:

| | fires? |
|---|---|
| `Process { running: true }` | **yes** |
| `Timer { onTriggered }` | **never** |
| `Process { stdout: StdioCollector { onStreamFinished } }` | **never** |

This is the single most useful fact here, and it is why an obvious test harness
silently produces nothing. A harness built on `Timer` or `onStreamFinished`
writes no file, reports no error, and looks exactly like a passing run.

**What does work, and what to test instead:**

- **`Process` execution is observable.** Point `bch-bot` at a logging shim on
  `PATH` and the app's own argv tells you what it did:

  ```bash
  #!/bin/sh
  echo "INVOKED: $*" >> /tmp/calls.log
  exec /home/luke/.local/bin/bch-bot "$@"
  ```

  ```
  PATH=/tmp/shim:$PATH quickshell -p /tmp/qs-test
  → /tmp/calls.log:  INVOKED: balance
  ```

- **To observe a QML object's own state**, have the object report it. Append a
  temporary probe to the widget — a `Process` writing `panelOpen` to a file on
  every change — reload, click, read the file. A QML child has no surface of
  its own, so `hyprctl layers` will never show it, and the object telling you is
  the only authoritative source. **Remove the probe before committing.**

- **Verify the CLI directly** for anything the UI merely calls. The panel shells
  out to `bch-bot`, so its flows are testable without the panel at all, and that
  is where real bugs were found.

### What this costs you

`INVOKED: balance` proves the panel loaded and started its refresh. It does
**not** prove the send picker, the swap chips, or the quote display work.

That said, the dead signals are no longer a wall: with a working screenshot
(§2) and a cursor on the bar (§3), a flow can be driven for real — click the
slot, capture, read the panel. These signals are the fallback for state a
screenshot cannot show, such as a property behind a hidden panel. Claiming a UI
"verified" on the strength of a startup command is overclaiming, and it is worth
being blunt about it.

## 5. A plugin is not deployed until the installed clone is updated

The install directory is a **git clone of the same repo you commit to**:

```bash
cd ~/.config/omarchy/plugins/io.github.lucasmcducas.bch-wallet
git fetch origin && git log --oneline HEAD..origin/main    # what you are missing
git cherry -v origin/main HEAD                            # is local work already upstream?
git pull --ff-only origin main                             # or: git reset --hard origin/main
```

`git cherry` is the safety check. A `-` prefix means the local commit is
patch-identical to something already upstream, so a `reset --hard` discards
nothing. Without that check, a divergent install makes `--ff-only` fail and the
obvious next step is a force, which is how real work gets lost.

The shell runs with `-n` (do not reload on config change), so **it must be
restarted** for new panel code to reach the screen:

```bash
export HYPRLAND_INSTANCE_SIGNATURE=<sig> XDG_RUNTIME_DIR=/run/user/1000 WAYLAND_DISPLAY=wayland-1
nohup quickshell -n -p /usr/share/omarchy/shell > /tmp/qs.log 2>&1 &
```

Use `kill <pid>` on a specific pid. **`pkill -u luke -f quickshell` matches your
own ssh session's command line and kills the connection mid-command** — that
cost two confusing empty results before it was spotted.

## 6. Opening the wallet panel: right click, and find the slot from `shell.json`

```qml
// BchBalanceWidget.qml
function handlePress(pressedButton) {
  if (pressedButton === Qt.RightButton) { root.panelOpen = !root.panelOpen; return }
  root.refresh()
}
```

Right click opens it; left click only refreshes. On a vertical bar the wallet slot
shares a 28px strip with every other omarchy widget, so the y that hits it is not
knowable from the widget's own source.

**The slot's position is derivable instead of sweepable.** `shell.json` holds the
bar layout — a `layout` per section, each an ordered array of plugin ids:

```json
"bar": {
  "center": ["system-update", "bluetooth", "network", "audio", "microphone"],
  "right":  ["tray", "io.github.lucasmcducas.bch-wallet", "agents", "power"]
}
```

In a **vertical** bar `right` maps to the lower half and array order runs top to
bottom. So read the array, count the rendered bands, click that y. On this
machine the strips with content were:

```
14-25  41-49  71-79  101-109  131-137  161-169      left + center
508-516  539  562-570  586-593                      right: tray, … , wallet, …
971-982  999-1008  1026-1036  1053-1063             bottom group
```

Detect the bands from pixels rather than by sweeping blind:

```python
a = np.array(Image.open(shot).convert("RGB").crop((0, 0, 28, 1080)))
rows = [y for y in range(1080) if a[y].mean(axis=1).max() > 100]
# group consecutive rows -> one run per rendered widget
```

### The plugin contract, and how failing it looks

The plugin dir is auto-scanned, so nothing is "registered" anywhere — but three
things have to line up or the widget silently does not exist:

- **`manifest.json` is required.** No manifest, no load. First-party examples sit
  in `/usr/share/omarchy/shell/plugins/bar/widgets/*.manifest.json`.
- **The root QML's `moduleName:` must equal the manifest `id`.** When it does not,
  the object still instantiates — so its background processes start, which reads
  as success — and nothing is drawn.
- **The id must appear in the `shell.json` layout array.** A `defaultSection` in
  the manifest does not put it in the bar.

And one subtlety that specifically breaks this widget's design: it gates drawing
on an async status,

```qml
readonly property bool slotVisible: status !== "no-wallet"
visible: slotVisible
```

`Component.onCompleted` fires regardless of `visible`, so a `bch-bot` process
running is proof of instantiation and **not** proof of visibility. Measure pixels
for the second.

## 7. hyprctl here is a Lua API, and there is no mouse click at all

Hyprland 0.56.2 on this machine does not accept the classic dispatcher commands.
`hyprctl dispatch movecursor 100 100` is fed to Lua and fails with
`'(' expected near '100'`. The interface is `hl.dsp.<group>.<fn>`:

```bash
hyprctl eval     'hl.dsp.cursor.move({x=14, y=566})'   # returns ok, DOES NOTHING
hyprctl dispatch 'hl.dsp.cursor.move({x=14, y=566})'    # ok — this is the one that works
```

That difference cost a long stretch of the verification. `hyprctl eval` reports
success and is inert; `hyprctl dispatch` with a full Lua expression works. I spent
many rounds concluding "input is broken" while sending only `eval` calls.

Groups present: `cursor.move`, `send_key_state`, `exec_cmd`, `exec_raw`,
`window.*`, `workspace.*`, `group.*`, `focus`, `dpms`, `pass`, `event`. **There
is no click or mouse-button dispatcher** — not `click`, not `moveactivepointer`,
nothing. Right-clicking a widget is not available on this box, by any route.

`send_key_state` does work, with `{ mods, key, state }` where `mods` is a
**string** and `state` is `"down"`/`"up"`/`"repeat"` (not `"pressed"`). Verified
by sending Escape to close `omarchy-menu`, which is a real effect on the desktop.

## 8. Driving a Quickshell panel with no mouse: IpcHandler

Since a click is impossible, a click-only widget is unreachable. The omarchy
plugins already solve this, and so does the wallet panel now:

```qml
IpcHandler {
  target: "bch-wallet-panel"
  function open() { root.panelOpen = true }
  function toggle() { root.panelOpen = !root.panelOpen }
}
```

Called with the **live instance pid** — bare `qs ipc call` fails with "Could not
find config directory" because the shell runs with `-p` and has no default config
name:

```bash
pid=$(for p in /proc/[0-9]*; do [ "$(cat $p/comm 2>/dev/null)" = quickshell ] && basename $p; done | head -1)
qs ipc --pid $pid call bch-wallet-panel open
```

**QML has no return-type syntax on functions.** `function open(): void` is a parse
error — I wrote nine of them and it took the entire panel down.

## 9. Restarting the shell, and where load failures hide

`quickshell -n -p …` refuses to start a second instance of the same config, and
`kill` returns long before the process has actually exited. Start too early and you
get:

```
An instance of this configuration is already running.
```

…with the new code silently never loaded. **Wait for the old instance to be gone
before starting the new one.** Related: `pgrep -f quickshell` matches the ssh
command line, so counts are always inflated and `pkill -f quickshell` kills the
session — enumerate `/proc/*/comm` instead.

When a widget fails to load, the shell does **not** stop, does not exit non-zero,
and prints nothing to stderr. The only record is the per-instance log:

```bash
d=$(ls -td /run/user/1000/quickshell/by-id/*/ | head -1)
grep -a -iE "failed|unavailable|is not a type" "$d/log.qslog" | tr -d '\000' | tail
```

It is interleaved with tens of thousands of lines of NetworkManager chatter, and
it contains **raw NUL bytes** — a plain read throws `UnicodeDecodeError`, so
decode with `errors="replace"`. This is where the parse error showed up as:

```
Plugin widget io.github.lucasmcducas.bch-wallet failed:
BchBalanceWidget.qml:249:5: Type BchWalletPanel unavailable
```

which reads like a wiring problem and is a syntax error 900 lines away. **Run
`qmllint <file>.qml` before deploying** — it is installed, it is instant, and it
would have caught it immediately.

## 10. What vision does and does not see on a 28px bar

Handed a downscaled full-screen shot, a vision model described the bar's contents
confidently and **confabulated the rest** — it reported reading a Bitcoin symbol
on a blurred icon that was the network glyph, and transcribed digits for widgets
that show no text. The bar is 28px wide; at that size the honest answer is
usually "illegible".

Use measurement for existence, and vision only for surfaces large enough to
actually read:

```bash
# the strip's real colours, on the remote box (ImageMagick is there, PIL is not)
convert shot.png -crop 28x1080+0+0 +repage -colors 6 -format '%c' histogram:info:
```

A large open panel *is* legible once the screen is unlocked. The path to getting
there: wake the screen, open the panel over IPC, capture `fullscreen save`, scp
the PNG, and read it.

One caution in the other direction: vision also **invents defects**. Asked about
the send view it reported a `bitcoincash:q…` placeholder as "truncated" — the
ellipsis is the intended text — and the asset selector as "empty" when
`sendAsset: "bch"` renders as a selected chip rather than a text value. Check the
code before believing a claimed bug.

## 11. Driving the UI the way a user does

Two bugs shipped here that only a real click could have found, and the second
one is the more interesting failure of method.

### The click did nothing, and no error said so

```qml
onPressed: root.handlePress
```

That passes a bare function reference, so QML calls `handlePress` with **no
arguments**. `pressedButton` is always `undefined`, so every branch in the
handler failed silently — a user clicking the wallet got no panel, no refresh,
and nothing in any log. The fix is the lambda form every first-party widget uses
(`Microphone.qml:45`):

```qml
onPressed: function (b) { root.handlePress(b) }
```

I did not catch this because I had been driving the panel over IPC, which
bypasses the click path entirely. **A control that only works when you do not
click it is not a control.** Verify the path a user takes, not a side door added
for testing.

Left click now opens the panel, as users expect. It was right-click only, with
left doing a refresh whose effect — a number changing — is invisible. Every
first-party widget acts on left click.

### Synthesising a real click on a build with no click dispatcher

`hl.dsp` has `cursor.move`, `send_key_state`, `exec_cmd` and `window.*`, and
**nothing for mouse buttons**. `wtype` types keys; `ydotool`/`xdotool` are not
installed. So a click needs a virtual device. The parts that are not obvious:

```python
# struct uinput_user_dev is 1116 bytes, and a plausible-looking wrong packing
# makes write(2) fail EINVAL:
#   name[80] + input_id{4 x u16} + ff_effects_max(u32) + 4 arrays of u32[64]
ABS_CNT = 64
buf  = bytearray(b"hermes-mouse" + b"\x00" * 60)
buf += struct.pack("<HHHH", 0x03, 0x1234, 0x5678, 0x0001)   # BUS_VIRTUAL
buf += struct.pack("<I", 0)                                  # ff_effects_max
buf += b"\x00" * (4 * ABS_CNT * 4)
assert len(buf) == 1116
```

- **Use RELATIVE axes.** `UI_SET_ABSBIT` fails EINVAL on this kernel, and
  absolute axes need a per-axis `UI_ABS_SETUP` absinfo before `UI_DEV_CREATE`.
  Relative axes need neither and match the real mice on the box.
- **`UI_ENABLE` is EINVAL for every code tried** (`REL_X=0`, `BTN_LEFT=0x110`).
  It looks like an unsupported ioctl rather than a bad argument. Buttons and
  relative axes both default to enabled, so treat it as best-effort — making it
  fatal meant the device never existed at all.
- **Write `SYN_REPORT` as a normal event**, not as eight raw NUL bytes, which is
  EINVAL. Writing it raw threw *partway through the glide*, after the cursor had
  already moved — so a working device looked like it failed at random.
- **Place the cursor with the compositor, not with the device.** Pointer
  acceleration means a relative delta of N does not move N pixels: one run wanted
  `14,566` and ended at `19,764` after six correction passes, and never emitted a
  click. That reads exactly like "the widget is not clickable".

```python
subprocess.run(["hyprctl", "dispatch", f"hl.dsp.cursor.move({{x={tx}, y={ty}}})"])
# then only the button press/release via uinput
```

`/dev/uinput` is root-only for luke, so run it as root:

```bash
env -i PATH=/usr/bin:/bin HOME=/home/luke ~/.hermes/bin/omarchy-sudo   python3 /tmp/click.py <x> <y> left
```

The clean env matters — the helper's venv otherwise fails with
`undefined symbol: PyType_GetName` from its `cryptography` import.

### The widget was at y=976, and pixel-diffing could not find it

Measuring which bar rows change when the panel opens over IPC said **562-570**,
and clicking there opened the **calendar**. The wallet is at **y=976**.

Pixel-diffing the bar tells you *what changed*, not *what is there* — two
widgets light up at once and neither diff is wrong. Only a log inside the widget
distinguishes them.

That log took four attempts, because every mechanism available in this Quickshell
build was dead, and each failure looked like "the click does not work":

| attempt | why it failed |
|---|---|
| `FileView` | `print` and `writeMethods` do not exist in 0.3.1 — both are hard load errors |
| `Process`, `running` false→true in one tick | QML collapses the change; it fires once at load and never again |
| `Timer` to space the two apart | `Timer.onTriggered` is a **dead signal** here |
| `Process` started the way `refresh()` starts `balanceProcess` | works |

A plain `Process` with a shell append, started the proven way, is what finally
showed `press button=seen-1` on a real click. It is not worth shipping, so it is
out — but it is what proved the click.

**That dead `Timer` is also a live bug in the widget.** Its 60-second refresh
Timer cannot tick, so the bar balance only updates when the panel is opened.
Separately worth fixing.

### QML traps that each took the whole panel down

Every one surfaces as `Type BchWalletPanel unavailable` — an error blaming a
*wiring* problem, pointing 120 lines away from the actual cause. The shell does
not stop, does not exit non-zero, and prints nothing to stderr; only the
per-instance log records it.

- **No return types on functions.** `function open(): void` is a parse error.
- **A `TapHandler` has no `anchors`.** It is a pointer handler, not an Item; it
  covers its *parent's* bounds. `anchors.fill` on one is
  `Cannot assign to non-existent property "anchors"`.
- **An `IpcHandler` has no default property**, so it cannot hold child objects.
- `qmllint` catches the return-type syntax and none of the other three — those
  are valid QML, invalid against this Quickshell version.

And one that is a design bug rather than a syntax error: a backdrop `TapHandler`
on a full-panel overlay covers the search field and every list row, so clicking a
token just closes the picker. A pointer handler is not a dismiss affordance.

## 12. What is actually verified

At plugin `60b193c`, CLI `3731be4`, on the machine, shell restarted, install clean:

| flow | how | result |
|---|---|---|
| bar widget renders | pixels | themed `#1A1B26` strip with widget content |
| **left click opens panel** | **real uinput click at y=976** | `BCH Wallet` / balance / `ROACH 2` / Receive Send Swap |
| home | screenshot | balance matches `bch-bot balance` exactly |
| send | screenshot | asset chips, recipient + amount, Preview / Confirm & send |
| swap view loads | screenshot | no false "no market" error |
| **receive: address resolves** | **real click on Receive** | no spinner left hanging; fresh CashAddr each time |
| **receive: QR renders** | screenshot + decode | 34 KB SVG on a white plate; `zbarimg` decodes it |
| **receive: QR == shown address** | decode vs on-screen text | **byte-identical** |
| **receive: Copy address** | real click, clipboard cleared first | `wl-paste` returns the same address |
| token search | CLI | `--search pusd`→PUSD, `--search roach`→ROACH (outside top 20), `--search 2469acc5`→PUSD by category |

The receive flow is verified end to end with real clicks, and the check that
matters is the one asserting **all three transports agree** — displayed text,
QR decode, and clipboard. Any one of them passing proves nothing on its own; see
[[references/receiving-an-address]] for why, and for the two bugs that got
through before that comparison existed.

**Not verified:** the Swap button and token picker have not been exercised with
real clicks end to end. The CLI search is proven and the picker is implemented,
but clicking through Swap → search → select has not been driven. And no broadcast
has been executed — deliberately, since it moves real value.

## 13. Log the invocation, not the result

The single most useful diagnostic on this box, and it took four wrong turns to
arrive at. A view was intermittently stuck on its loading text; the cause was
either a command not being dispatched or a command being dispatched and failing,
and **the UI looks identical in both cases**. Reading the result-handling code
could not separate them.

```sh
echo "$(date +%s.%N) $*" >> /tmp/bch-calls.log     # in the bch-bot shim
```

An **empty** log means never dispatched. A line means it ran, and the failure is
downstream. It took one `wc -l` to settle a question that several rounds of
screenshot-reading had not. Keep a copy of the shim and remove the line
afterwards.

This is the same "assert on the difference, not on presence" lesson as §11, at
a different altitude: there the question was *did the UI change*, here it is
*did the command run*. When the two candidate causes need opposite fixes, find
an instrument that reads the thing they disagree about rather than a proxy for
it.

## 14. `qs ipc` needs `-p`, and it discards return values

Two independent ways to conclude "the panel is dead" that are both wrong:

- Without `-p /usr/share/omarchy/shell` it dies with `Could not find "default"
  config directory`, because the shell runs with `-p` and has no default config
  name.
- **It returns empty for every function**, including ones that plainly return a
  string — `qs ipc show` types them all as `(): void`. Empty output means
  nothing at all; the handler may be perfectly healthy.

```bash
qs ipc -p /usr/share/omarchy/shell show            # the subcommand is `show`, not `-l`
qs ipc -p /usr/share/omarchy/shell call <target> <fn>
```

`show` prints every registered target and function, which is the fastest way to
confirm a handler exists. Because return values are dropped, assert on side
effects — a file, a log, a screenshot.

**An IpcHandler view switch is not the button it mimics.** A `receive()` handler
that sets `root.view = "receive"` looks like the Receive button and does not call
`loadAddress()`; only the button's `onClicked` does. Driving the handler proved
the view renders and the command never runs. That one-line difference is the
shape of the entire §13 bug, and it is invisible unless you read what the button
actually calls.

## The lesson

Every one of these is an **environment** fact, and every one produced a wrong
conclusion first:

| first conclusion | what was actually true |
|---|---|
| "qs.Ui is not installed, so the panel cannot load" | it is in `/usr/share/omarchy/shell` |
| "screenshots are impossible on this display" | `omarchy capture screenshot fullscreen save` works; only `grim` hangs |
| "the bar renders nothing, so the widget is broken" | the screensaver was covering the screen |
| "input is broken, clicks do nothing" | `hyprctl eval` is inert; `hyprctl dispatch` with a Lua expression works |
| "I can right-click the widget" | this build has no click dispatcher at all — IpcHandler instead |
| "the panel runs `bch-bot`, so the UI works" | only a startup `balance` was observed |
| "the panel is deployed" | the installed clone was four commits behind |
| "the framebuffer is blank" | `dd` was permission-denied on a root-only device |
| "a panel opened at y=80" | the wallpaper is animated and moved |
| "the send view has a truncated placeholder and an empty selector" | the ellipsis is intended; the chip is selected |
| "the router has no live market for BCH↔pusd" | it does — 3.15 PUSD across 12 pools; the panel's read failed |
| "clicking the widget does nothing" | `onPressed: root.handlePress` passes no button, so every branch silently failed |
| "the widget is at y=562-570" | it is at y=976; the bar diff lit two widgets and neither diff was wrong |
| "the receive view is stuck, so the result handler is broken" | the command was never dispatched — an invocation log was empty |
| "the Copy button is broken, the clipboard came back empty" | a leftover `wl-copy` from an earlier test still owned the buffer |
| "`qs ipc` returned nothing, so the handler is dead" | this build discards return values; the handler was fine |

Seven patterns generalise:

**A single tool failing is not a property of the machine.** `grim` hanging says
something about `grim`; `hyprctl eval` doing nothing says something about
`eval`. Before recording a capability as absent, check what else the system
provides — and if the claim is already written down, correct it in place, keeping
the wrong version visible. A wrong fact in a wiki is worse than a missing one:
it stops the next reader from checking.

**A live process is not a visible surface, and a success code is not a success.**
The widget ran `bch-bot` perfectly while hidden behind a lock. `hyprctl eval`
returned `ok` while doing nothing. `quickshell` started, printed no error, and
exited zero with the panel's parse error silently swallowed. Every one of those
needed a **positive observation** — pixels, a cursor position, a file that
should exist — to tell the truth.

**Establish the baseline before trusting a surprising result.** `hyprctl clients`
answers "is there even an unlocked desktop here" in one call. It should have been
the second thing I ran, not the fortieth — and the black bar in the first capture
was already the evidence.

**Verify the path a user takes, not a side door you added.** The wallet panel
was fully working over IPC and completely dead to a real click, for an entire
session, because I only ever drove it the way I had made it drivable. Any
affordance reachable only through your own test harness is a control that has not
been tested.

**When your instrumentation cannot see something, suspect the instrument first.**
Four separate mechanisms — FileView, a restarted Process, a Timer, a pixel diff —
each reported "nothing happened" and each was itself broken. A diagnostic that
says nothing is not evidence of absence.

**A message on a screen is a symptom until you check the thing it names.** The
panel said the router had no market, and that is a testable claim about the
world, not about the panel. One command — `bch-bot swap BCH pusd 0.01` — settled
it, and it should have been the first thing I ran rather than something Luke had
to correct me on. A UI that reports an external condition is making a claim; the
cheapest way to respect it is to ask the thing it is claiming about.

**A loading state with no error is a dropped request until proven otherwise.**
The shared-process helper guarded itself with `if (status === "busy") return` —
sound as mutual exclusion, and wrong in a panel that refreshes in the background,
because "busy" is the *normal* state when a user clicks. The request was thrown
away roughly one click in three and nothing retried it. Queue it. This is the same
shape as §11's dead signals: a defensive guard that is wrong about which state is
normal turns a transient collision into a permanent hang with nothing to see.

**Verify the cheap thing before the expensive thing.** `qmllint` on a QML file is
instant and would have caught a parse error that cost a deploy, a restart, and a
dig through 60KB of log. Reach for the local linter before the remote round trip,
not after.

## See also

- [[references/in-wallet-swaps]] — the swap engine this panel drives
- [[references/swaps/gates-that-pass-when-they-should-fail]] — the same discipline
  applied to tests
