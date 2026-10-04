# Driving the omarchy box

What it takes to see, click, and verify a Quickshell UI on the omarchy machine —
and the three things that had to be discovered the hard way before any of it
worked.

This is a record of environment facts, most of which are not discoverable from
documentation and all of which cost time to find.

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

## 2. `grim` cannot screenshot this display, and it is not a permissions problem

Screenshots hang. Every variant was tried: bare `grim`, `-o` with every
plausible output name, `systemd-run --user`, `uwsm app`, and with
`HYPRLAND_INSTANCE_SIGNATURE` both set and unset. Some fail fast with a real
error; the ones that matter **connect and then block forever**.

The discriminator is instructive:

```
WAYLAND_DISPLAY unset   →  grim: "failed to create display"   (instant)
WAYLAND_DISPLAY=wayland-1 → grim connects, then blocks        (timeout)
```

So the compositor accepts the client and the *capture* is what fails. The cause
is the display: the DRM connector is driven by **`simple-framebuffer`**, a virtual
framebuffer with no scanout engine, so `wlr-screencopy` has no buffer to hand
back. A screenshot is not possible here, and no screenshot tool will change that.

`hyprctl layers` is the substitute for *seeing*. It reports the real surface
tree, which is how the 28px bar and the 1920×1080 background were identified.

**Consequence:** visual verification of a Quickshell UI on this box is
impossible. Verify behaviour instead — see §4.

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
**not** prove the send picker, the swap chips, or the quote display work — none
of those can be triggered without a signal that fires. Claiming a UI "verified"
on the strength of a startup command is overclaiming, and it is worth being
blunt about it.

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

## 6. Opening the wallet panel: right click, and the position is not obvious

```qml
// BchBalanceWidget.qml
function handlePress(pressedButton) {
  if (pressedButton === Qt.RightButton) { root.panelOpen = !root.panelOpen; return }
  root.refresh()
}
```

Right click opens it; left click only refreshes. On a vertical bar the wallet
slot shares a 28px strip with other omarchy widgets, so the y that hits it is
not knowable from the source and has to be swept.

## The lesson

Every one of these is an **environment** fact, and every one produced a wrong
conclusion first:

- "qs.Ui is not installed, so the panel cannot load" → it is in
  `/usr/share/omarchy/shell`
- "the display is missing" → it is a `simple-framebuffer` with no scanout
- "the panel runs, so the UI works" → only a startup `balance` was observed
- "the panel is deployed" → the installed clone was four commits behind

The pattern: **when a check on a remote machine reports something surprising,
establish the machine's baseline before trusting the check.** `hyprctl layers`
answers "is there a desktop and what is on it" in one call, and it should have
been the second thing I ran, not the fortieth.

## See also

- [[references/in-wallet-swaps]] — the swap engine this panel drives
- [[references/gates-that-pass-when-they-should-fail]] — the same discipline
  applied to tests
