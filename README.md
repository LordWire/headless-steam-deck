# steamremote — headless Steam on Fedora for Steam Deck Remote Play

![Vibe Coded](https://img.shields.io/badge/vibe_coded-100%25-a855f7?style=for-the-badge)

Stream games from a Fedora desktop to a Steam Deck over Remote Play, **without
logging in at the machine** and **without root**.

`steamremote` starts Steam inside [gamescope](https://github.com/ValveSoftware/gamescope)
in a headless session, so the GPU is used for real rendering while your monitor
keeps showing whatever it was already showing. One command starts it, one command
stops it, and a forgotten session kills itself.

```
ssh myhost
steamremote            # Steam is now up and streamable
# ... connect from the Deck ...
steamremote stop
```

## Why this exists

This is part of a greater project, that attempts to switch on a machine and initiate gamescope over ssh in one step.
For now it's just handling gamescope, in the future it will also be remotely invoked via a deck plugin.
Currently this is heavily vibe coded, so expect bugs and weird docs.

## Requirements

- Fedora (tested on **Fedora 44**, KDE Plasma 6, Wayland)
- **A single AMD GPU on the open driver** — tested: Radeon RX 7600M XT, Mesa 26 /
  RADV. Nothing here is AMD-specific, but the encoder step below is written for
  Mesa, and headless streaming has open upstream bug reports under NVIDIA (see
  [Limitations](#limitations-what-is-not-covered)). If you are on NVIDIA with a
  hybrid/multi-GPU box, read that section before you start.
- A Steam Deck already paired to the same account.
  normal graphical login once; after that it persists.

## Install

1. Enable RPM Fusion (needed for Steam and for hardware video encoding):

   ```
   sudo dnf install https://mirrors.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm \
                    https://mirrors.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm
   ```

2. Install the packages:

   ```
   sudo dnf install gamescope steam mesa-va-drivers-freeworld libva-utils
   sudo dnf install mesa-va-drivers-freeworld.i686
   ```

   | Package | Provides | Needed for |
   |---|---|---|
   | `gamescope` | Fedora repo | the headless compositor |
   | `steam` | RPM Fusion nonfree | the client itself |
   | `mesa-va-drivers-freeworld` (+ `.i686`) | RPM Fusion free | **H.264/HEVC hardware encode** |
   | `libva-utils` | Fedora repo | `vainfo`, to check the encoder |

   The freeworld Mesa build matters: Fedora's stock Mesa ships with H.264 and
   HEVC *encoding* disabled for patent reasons. Without it Steam silently falls
   back to a 4-thread software `x264`, and the stream stutters on anything modern.

   Verify the encoder is visible:

   ```
   vainfo 2>/dev/null | grep -E "H264.*ENCODE|Main.*Profile"
   ```

   You want to see `VAEncH264EncodeEntry` / `SAH264_ENCODER` profiles. If you see
   nothing, the freeworld driver is missing or the `.i686` copy is absent.

3. Install the script:

   ```
   install -m755 steamremote ~/.local/bin/steamremote
   ```

4. Start Steam once from your normal desktop and log in. The headless session
   cannot type a password, and it reuses the stored login.

## Usage

```
steamremote                 # start; 1280x1024@60; auto-stops after 2h
steamremote launch 90m      # start with a different limit
steamremote launch none     # no time limit
steamremote status          # running? is Remote Play reachable?
steamremote logs            # follow the session log
steamremote stop            # stop the whole session
```

Configuration by environment variable:

```
GAMESCOPE_WIDTH=1920 GAMESCOPE_HEIGHT=1080 GAMESCOPE_REFRESH=60 steamremote launch
GAMESCOPE_CAP=6h steamremote
```

State lives in `~/.local/state/steamremote/` (`gamescope.pid`, `run.log`).
Remote Play advertises on UDP/TCP 27036.

## How it works

`steamremote` runs one detached command:

```
setsid nohup timeout -k 30 <cap> gamescope \
    --backend headless \
    -W 1280 -H 1024 -w 1280 -h 1024 -r 60 \
    --steam \
    -- steam -tenfoot -gamepadui -pipewire -pipewire-dmabuf >> run.log 2>&1 </dev/null &
```

Four details that are not obvious, and each one is load-bearing:

1. **`--backend headless`** — gamescope renders on the GPU without touching DRM
   or a virtual terminal. This is what removes the need for root.
2. **`setsid`** — the session becomes its own process group, so closing the SSH
   connection cannot signal it.
3. **The PID file stores the session leader**, which is also the process-group ID.
   `stop` sends one signal to `-PID` and it reaches `timeout`, gamescope, the
   reaper, Steam and the running game at once. Nothing is orphaned.
4. **`</dev/null` on stdin** — Steam reads stdin, and without this redirect the
   launcher blocks waiting for a terminal.

### Built-in guards

- **Refuses to start if your desktop session is running** (looks for
  `kwin_wayland` as your user), so it never fights your local login for the GPU.
- **Refuses to double-start.**
- **Auto-stops after a time cap** (default 2h) via `timeout -k 30`, so a forgotten
  session cannot hold the machine forever.
- **Self-heals a stale PID file** — a session that ended via the cap, a crash or
  an OOM kill leaves the file behind, so `status` verifies the PID really is
  gamescope by reading `/proc/<pid>/cmdline` rather than trusting the number.
  Without this, a recycled PID would block every later launch.

## Troubleshooting

**Black or frozen picture on the Deck.** Check the Steam-side log for the capture
and encoder it negotiated:

```
grep -iE "capture method|encoder|libx264|vaapi" ~/.local/share/Steam/logs/streaming_log.txt | tail
```

`libx264` means the hardware encoder was not found — go back to the `vainfo`
check above.

**Steam not logged in / empty Big Picture.** The headless session cannot enter
credentials. Log in once from a normal desktop session first.

**Won't start, says your desktop session is running.** That is the guard working.
Log out of the desktop (not just lock it) — the session has to actually end.

**Leftover sockets after `kill -9`.** Only if you killed it yourself:

```
rm -f /run/user/$(id -u)/gamescope-*
```

**Audio.** See below — behaviour differs from what people expect.

## Audio notes

The game's audio is *copied* into the stream, not diverted from the host — so in
general you should expect to hear it locally too, and mute the host sink if you
don't want that.

On a headless session started over SSH, though, you will probably hear nothing at
all, for an unrelated reason. Linux sound devices are gated by `systemd-logind`
to the **active** session, and an SSH session is not active:

```
$ ls -l /dev/snd/controlC0
crw-rw----+ 1 root audio 116, 7 ...          # note the '+' = an ACL is set
$ getfacl -p /dev/snd/controlC0
user:sddm:rw-                                # granted to the greeter, not to you
other::---
$ wpctl status | sed -n '/Sinks/,/Sources/p'
 33. Dummy Output                            # no real device reachable
```

So PipeWire has no hardware sink to write to, falls back to a null "Dummy Output",
and the host stays silent. Whether the Deck still receives audio depends on Steam
encoding from that stream rather than from the device — I have not verified it, so
test it before relying on it.

If you want real audio routing in the headless session, the two clean options are
to give the session its own PipeWire instance with a virtual sink, or to add your
user to the `audio` group — the latter grants permanent access to every sound
device on the machine, so it is a deliberate trade, not a convenience.

## Removing it

```
rm ~/.local/bin/steamremote
rm -rf ~/.local/state/steamremote
```

Nothing else was written, and nothing runs at boot.

## Limitations (what is not covered)

- **NVIDIA or Intel GPUs — untested, and partly known-broken.** Every upstream
  bug report about this backend is an NVIDIA or hybrid/multi-GPU host:
  [gamescope#1984](https://github.com/ValveSoftware/gamescope/issues/1984) is a
  Fedora box with GTX 1660 SUPER + RX 7600 where the headless session streams
  audio but the client sees **only a cursor**;
  [gamescope#1596](https://github.com/ValveSoftware/gamescope/issues/1596) is
  black video to a Deck under NVIDIA, root cause a missing
  `VK_IMAGE_USAGE_SAMPLED_BIT`, worked around by launching Steam with
  `-pipewire` — which this script already does. Single-GPU AMD/RADV is the one
  configuration actually verified working.
- Running alongside an active local desktop session — by design these are
  mutually exclusive
- System-wide or multi-user installation
- Any guarantee about latency or framerate; with no DRM output to read from, Steam
  uses the PipeWire desktop-capture path rather than the per-game NV12 path

## Prior work

This project attempts to optimize the lifecycle of gamescope and improve the experience of
game streaming on other devices (namely, the Deck).

the giants whose shoulders I stood on in order to figure out what's there and what's missing: 
- [Steam Remote Play Headless — bwc9876.dev](https://bwc9876.dev/blog/headless_steam_remote_play/)
  — the same `--backend headless --steam -- steam -tenfoot -pipewire-dmabuf`
  invocation, on NixOS, kept alive by running it inside GNU `screen`.
- [A Steam Deck Everywhere — blog.doomsdayrs.page](https://blog.doomsdayrs.page/a-steam-deck-everywhere)
  (2024-08-23) — a launch-only script, with useful per-flag commentary
  (`-e`, `-o 1`, `--prefer-vk-device` for multi-GPU boxes).
- [Linux remote game streaming — nelsonslog](https://nelsonslog.wordpress.com/2026-07-12/linux-remote-game-streaming/)
  — a survey of the routes and why each one is fragile; good context on the
  dummy-plug and EDID-spoof alternatives this avoids.

Adjacent projects that solve the same goal with different technology, worth
knowing before you commit to Steam's own protocol:

- [Moonshine](https://github.com/hgaiser/moonshine) — spins up its own Wayland
  compositor per stream and serves the Sunshine/Moonlight protocol; lower
  latency than Remote Play, but it essentially is a full remote desktop which I didn't want
- [docker-steam-headless](https://github.com/Steam-Headless/docker-steam-headless)
  — containerised Xorg + VNC host. Heavier, and not gamescope-headless.
- [Sunshine's headless-over-SSH guide](https://docs.lizardbyte.dev/projects/sunshine/v0.23.0/about/guides/linux/headless_ssh.html)
  — the same goal via X plus a fake display.

Fedora-specific encoding gotchas

- [Fedora RPM Fusion Multimedia howto](https://rpmfusion.org/Howto/Multimedia)
  — why stock Mesa has no H.264/HEVC *encode*.
- [steam-for-linux#11028](https://github.com/ValveSoftware/steam-for-linux/issues/11028)
  — `vainfo` looks perfect on 64-bit while Steam still fails, because Steam is a
  32-bit process: `radeonsi_drv_video.so: wrong ELF class: ELFCLASS64`. This is
  the reason `.i686` is in the install list above, and the reason a stream can
  stutter for no visible cause.

## License

MIT — do whatever you like with it, just keep the copyright notice. See
[LICENSE](LICENSE).
