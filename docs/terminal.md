# Terminal Stack

## System Overview
- **Terminal emulator:** roxterm 3.10.1 (GTK3, libvte-2.91 0.70.6)
- **Console font:** Terminus Bold 32x16
- **Framebuffer terminal:** fbterm 1.7 (optional, see caveats)
- **Panel:** 3840x2160 eDP-1, X runs at 1920x1080 via `--scale .5x.5`

The files below live outside `$HOME` and are therefore **not** tracked by this
repository. They must be reapplied by hand after a reinstall.

## roxterm: use the GTK3 build, not GTK2

antiX ships `roxterm-gtk2`, which links `libvte9` 0.28.2 — a terminal engine
from 2011. Old VTE prints the bytes of any CSI sequence its match table cannot
parse, so modern TUIs leak fragments like `[0>` on screen. It also hardcodes
`TERM=xterm` (8 colors).

```bash
sudo apt install roxterm    # 3.10.1-1~antix1, pulls libvte-2.91
```

Verify the engine rather than trusting the version string:

```bash
ldd /usr/bin/roxterm | grep vte    # want libvte-2.91.so.0, not libvte.so.9
```

VTE 0.70 exports `TERM=xterm-256color` and gives 256 colors. It still lacks the
kitty keyboard protocol, so Shift+Enter in Claude Code needs `/terminal-setup`.

## Console font

`/etc/default/console-setup` requests `TerminusBold 32x16`, but the cached boot
script can fall out of sync and load something else entirely. Check what
actually runs:

```bash
grep setfont /etc/console-setup/cached_setup_font.sh
sudo setupcon --save-only     # regenerate from /etc/default/console-setup
```

Font size can be changed live on a TTY; the change is not persistent.

```bash
setfont Uni2-TerminusBold24x12   # sizes: 18x10 22x11 24x12 28x14 32x16
```

## Emoji in a TTY: not possible

Kernel console fonts are 1-bit PSF bitmaps — color is a per-cell attribute, not
per-pixel — and the kernel caps a font at 512 glyphs (see the `charcount`
comment in `/usr/include/linux/kd.h`). Both installed fonts already sit at 512
glyphs with zero coverage above U+FFFF. No font or `TERM` setting changes this.

## fbterm: nicer rendering, unusable for TUIs

`fbterm` draws on `/dev/fb0` at native 4K with TrueType fonts, so text looks
sharper than anything in the scaled X session. Its terminal emulator is from
2010 and fails the same way VTE 0.28 did — `0q` and similar fragments leak from
unhandled sequences, and its own terminfo has no `Ss`/`Se`, `BE`/`BD` or `Smulx`.

It is also unable to render Nerd Font icons and box-drawing correctly at the
same time: `U+F0C2` (icons) and `U+2500` (box drawing) are both East Asian
Ambiguous, and `-a` is a single global switch for that class. Either icons are
clipped, or `tree` and `htop` alignment breaks.

**Use roxterm for TUI work.** Keep fbterm for casual console reading only.

```bash
sudo apt install fbterm fonts-symbola
sudo setcap 'cap_sys_tty_config+ep' /usr/bin/fbterm   # run without root
```

Launch via `~/.local/bin/fbt`, which forwards options (`fbt -s 32`) and sets
`TERM=fbterm` for 256 colors — fbterm otherwise hardcodes `TERM=linux` at 8.

## Display scaling caveat

`~/.screenlayout/default.sh` renders X at 1920x1080 and upscales 2x to the
panel, so glyphs carry half the detail the display can show. Native rendering
would need `--scale 1x1` plus `Xft.dpi: 192`, `GDK_SCALE=2` and a larger i3 bar
font. Deliberately not done — the scaled setup is simpler and apps that ignore
Xft DPI stay usable.
