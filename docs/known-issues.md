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

- **Session:** IceWM 3.6.0 (X11)
- **Display server:** Xorg

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
