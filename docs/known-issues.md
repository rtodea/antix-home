# Known Issues — Lenovo ThinkPad P73

## Hardware

- **Model:** Lenovo ThinkPad P73 (20QRS00200)
- **CPU:** Intel Core i9-9880H @ 2.30GHz (8 cores, normally 16 threads)
- **RAM:** (check with `free -h`)
- **GPUs:** Intel UHD 630 (integrated) + NVIDIA Quadro RTX 4000 Mobile / Max-Q

## CPU: Only 1 Core/Thread Active

The BIOS reports 8 physical cores but **Thread Count: 1**. The OS sees only a single CPU (`cpu0`). SMT control shows "on" but reports inactive — irrelevant since only one core is exposed in the first place.

```
CPU(s):              1
Core(s) per socket:  1
Thread(s) per core:  1
```

This is not a software/kernel configuration — there is no `nosmt` or `maxcpus=` in the boot parameters. The BIOS/firmware is presenting only 1 thread to the OS. This points to a **hardware failure or BIOS-level lockdown** (possibly triggered by a failing core causing machine-check exceptions).

### Impact
- Effectively running on a single-core machine (~2.3–4.8 GHz turbo)
- Severely limited multitasking and compilation performance
- Some CPU vulnerability mitigations note "SMT disabled"

### Possible next steps
- Check BIOS settings for "Active Processor Cores" or similar
- If BIOS shows all cores enabled, this likely indicates a hardware defect (CPU or mainboard)

## GPU: NVIDIA Quadro RTX 4000 Appears Idle

The system uses **Intel UHD 630** for all rendering (confirmed via `glxinfo`). The NVIDIA GPU is powered on but doing essentially nothing:

```
Power:   5W / 30W cap
Memory:  5MiB / 8192MiB
Util:    0%
Temp:    39°C
```

Only Xorg holds a minimal 4MiB allocation. No applications are offloaded to it.

### Fan observation
The thinkpad sensors report two fans running (2382 RPM, 1575 RPM) — these are the system/CPU fans. The GPU at 39°C and 5W would not spin a dedicated fan (if one even exists separately on this model).

### To use the NVIDIA GPU
See [graphics.md](graphics.md) for PRIME offload commands. The GPU is functional but unused by default in this hybrid/Optimus setup.

## Current Desktop Environment

- **Session:** i3 4.22 (X11) — switched from IceWM in July 2026
- **Display server:** Xorg
- **Config:** `~/.config/i3/config` (tracked in the home git repo)

The `.icewm/` configuration is still in the repo but is no longer used.

## Display: 4K Panel Run at 1920x1080

The internal panel (`eDP-1`) is natively **3840x2160**, but the session runs it
at **1920x1080**.

**This is the only thing making the desktop readable.** There is no `Xft.dpi`,
no GDK/Qt scale factor and no compositor scaling anywhere in the setup — the
readable size comes entirely from the reduced mode. If `eDP-1` ever reverts to
its native resolution, everything becomes unreadably small, and the fix is the
resolution, not the font sizes.

Nothing on disk used to set this, so it survived only as long as the running X
session did. It is now pinned by `~/bin/pin-internal-resolution`, wired into
the i3 config with `exec_always` so it re-applies on every start and reload:

```bash
xrandr --output eDP-1 --mode 1920x1080
```

Note `--mode` rather than `--auto`: `--auto` selects the *preferred* mode,
which is the native 4K being avoided.

The script is a no-op when `eDP-1` is switched off, so it does not undo
`~/bin/show-on-external-display-only`.

### Fonts and cursor at this resolution

Two settings exist because of the panel, and are easy to mistake for scaling:

| Setting | Where | Value |
|---------|-------|-------|
| i3 bar + window title font | `~/.config/i3/config` | `FiraCode Nerd Font Mono 11` |
| Cursor size | `~/.Xresources`, `~/.xsessionrc`, `gtk-3.0/settings.ini` | `48` |

Cursor size must be set in all three places: GTK/Qt/Chrome read `XCURSOR_SIZE`
from the environment at startup, **not** the X resource database, so
`~/.Xresources` alone leaves most applications at the 24px default.

### Gotcha: do not put "Noto Color Emoji" in an i3 font list

i3 previously used `pango:monospace, Noto Color Emoji 8`, which clipped the
text in the bar and in window titles. Noto Color Emoji is a bitmap-only colour
font, so pango reports inflated ascent/descent for it; i3bar sized the bar from
those metrics but rendered with the primary font. Emoji still resolve through
normal fontconfig fallback, so the family never needs to be listed explicitly.

### Gotcha: dex ignores AutostartCondition

The Magnus screen magnifier used to launch over the desktop on every i3 start.
Its entry in `/etc/xdg/autostart/` is guarded by an `AutostartCondition`
GSettings key that is set to `false`, but **dex** — which i3 uses for XDG
autostart — does not implement `AutostartCondition` and ran it regardless.
MATE/IceWM honour the condition, which is why it only appeared under i3.

Suppressed with a `Hidden=true` override at
`~/.config/autostart/magnus-autostart.desktop`, which takes precedence over
`/etc/xdg/autostart` without touching the packaged file. Delete that file to
re-enable the magnifier. The same trick works for any other unwanted entry in
`/etc/xdg/autostart` (`onboard`, `blueman`, `clipit`, …).

## Note on Omarchy / Hyprland (DHH's Desktop)

DHH (David Heinemeier Hansson) created [Omarchy](https://world.hey.com/dhh/omarchy-is-out-4666dd31), an opinionated Arch Linux setup using the **Hyprland** tiling compositor (Wayland-only). It's a sister project to his earlier [Omakub](https://omakub.org/) (Ubuntu + GNOME).

### Compatibility concerns for this machine

| Concern | Detail |
|---------|--------|
| **Wayland + NVIDIA** | Hyprland is Wayland-only. NVIDIA on Wayland historically has issues, though driver 535+ improves things. Would need `nvidia-drm.modeset=1` and env vars. |
| **Single-core CPU** | Hyprland + Wayland compositing on 1 core would be very tight. Animations and effects may stutter. |
| **Arch Linux** | Omarchy targets a fresh Arch install. Running Hyprland standalone on antiX/Debian is possible but not trivial (needs wlroots, wayland-protocols, etc. from newer versions than bookworm ships). |
| **Hybrid GPU** | Would need to decide: run Hyprland on Intel (simpler) or NVIDIA (better perf, more setup). |

### If you want to try Hyprland on this system
1. Easiest path: install on a separate Arch partition using `archinstall` with Hyprland profile
2. On current Debian: would require backports or building from source (Hyprland needs newer wlroots than bookworm provides)
3. Given the single-core limitation, expect reduced performance compared to a healthy system
