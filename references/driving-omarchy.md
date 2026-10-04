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

## 11. What is actually verified

Confirmed on the machine, at plugin `a8afe54`, with the shell restarted and the
widget in `shell.json`'s `right` group:

| view | rendered | notes |
|---|---|---|
| bar widget | yes | bar strip paints themed `#1A1B26` with widget content |
| panel opens | yes | `omarchy-keyboard-panel` layer appears via IPC |
| home | yes | `0.01658402 BCH`, `5 UTXOs`, `ROACH 2`, Receive/Send/Swap |
| send | yes | asset chips, recipient + amount fields, Preview / Confirm & send |
| receive | yes | `deriving…` while the address is computed, Copy address / Done |
| swap | yes | Sell/Buy tabs, BCH→pusd, quote controls — **and a real error** |

The balance matches `bch-bot balance` exactly, so the panel is displaying real
wallet state rather than a placeholder.

**The one genuine failure is the swap view**, and it is a real one rather than a
rendering artefact: the panel reports

```
no tokens have a live Cauldron market
```

in red. The swap flow is built and reachable; the router has no live market for
BCH↔pusd, so quotes cannot be fetched. That is a backend/liquidity condition, not
a UI bug, and it is the outstanding item before swap can be called working.

**Not yet verified:** a broadcast. No send or swap has been executed, so the
confirm-and-broadcast path is unexercised — deliberately, since it moves real
value. Getting there would need the swap market to exist first.

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

Four patterns generalise:

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

**Verify the cheap thing before the expensive thing.** `qmllint` on a QML file is
instant and would have caught a parse error that cost a deploy, a restart, and a
dig through 60KB of log. Reach for the local linter before the remote round trip,
not after.

## See also

- [[references/in-wallet-swaps]] — the swap engine this panel drives
- [[references/gates-that-pass-when-they-should-fail]] — the same discipline
  applied to tests
