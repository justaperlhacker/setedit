# SETEdit — modern build (revival)

SETEdit is a character-mode text editor with a 90s-style TUI, originally
written in 1996 by Salvador E. Tropea and built on Turbo Vision. This tree is
the canonical SourceForge suite (`setedit/code`: the editor, `infview`, plus
the `AlCon` and `cal` tools), revived so the **editor compiles and runs on a
modern x86-64 Linux toolchain** (tested with gcc 16.2.1 on CachyOS, as a native
**X11** application).

See `REVIVAL-PLAN.md` for the current status, the fixes required, and the
verification results. For a one-shot bootstrap of a fresh machine, use
`setup-new-host.sh.txt`.

## Source repositories

The revival reuses the RHIDE Turbo Vision stack. Clone as siblings:

| repo | remote | branch | role |
|------|--------|--------|------|
| setedit | `justaperlhacker/setedit` | `master` | editor + infview (this repo) |
| tvision | `justaperlhacker/tvision` | `modern-gcc` | Turbo Vision fork (gcc/glibc fixes) |

```sh
mkdir -p ~/Projects && cd ~/Projects
git clone git@github-justaperlhacker:justaperlhacker/setedit.git
git clone https://github.com/justaperlhacker/tvision.git
git -C tvision checkout modern-gcc
```

The build expects this sibling layout by default.

## Dependencies

- `gcc`, `g++`, `make`, `perl`, `git`
- ncurses, gpm, X11 and Xmu development files; zlib, bzip2, pcre
  (system copies are detected and used; the tree also ships fallbacks)

On Arch/CachyOS:

```sh
sudo pacman -S --needed base-devel ncurses gpm libx11 libxmu zlib bzip2 pcre perl git
```

## Build

See [`BUILD.md`](BUILD.md) for the full step-by-step instructions, build
variants (editor-only, `libset` for RHIDE), and troubleshooting. The short
version is below.

### 1. Turbo Vision (static)

```sh
cd ~/Projects/tvision/tvision
./configure --without-dynamic
make -j"$(nproc)"
ln -sf "$PWD/rhtv-config" ~/.local/bin/rhtv-config
```

### 2. SETEdit editor

```sh
cd ~/Projects/setedit/setedit
./configure \
  --tv-include="$HOME/Projects/tvision/tvision/include" \
  --tv-lib="$HOME/Projects/tvision/tvision/makes"
make -j"$(nproc)"
```

Outputs:

- `makes/editor.exe` — the editor
- `makes/infview.exe` — the InfoView file viewer

For the RHIDE shared library instead (unchanged interface):

```sh
./configure --libset --no-infview \
  --tv-include="$HOME/Projects/tvision/tvision/include" \
  --tv-lib="$HOME/Projects/tvision/tvision/makes"
make needed
make libset
```

## Running

The editor links Turbo Vision's X11 driver and picks it whenever `DISPLAY`
connects (otherwise it uses the terminal driver). It **must** be told where it
is installed:

```sh
# own X11 window
env DISPLAY=:0 SET_FILES=/usr ./makes/editor.exe

# terminal driver instead
env -u DISPLAY SET_FILES=/usr ./makes/editor.exe

# install system-wide (prefix /usr); installs the editor as "setedit"
make install
```

The X11 window sets `WM_CLASS = "tvapp", "SETEdit"`; find it with
`xdotool search --class tvapp` (its title is `SETEdit v0.5.8 ...`).

## Fixes vs upstream

Only two changes were needed, both in `setedit/config.pl`:

1. On UNIX the TV link libraries are taken from `rhtv-config --slibs` (not
   `--dlibs`). `--dlibs` returns only `-lrhtv`, which is correct only with a
   shared `librhtv.so`; the static-only TV fork needs `--slibs`, which also
   carries ncurses/gpm/X11/Xmu via `-Wl,-dn/-dy`.
2. The generated `infview` target depends on `needed`, so a parallel
   `make -j` cannot link `infview` before `libmpegsnd.a` exists.

## Notes / troubleshooting

- **`Wrong installation! You must define the SET_FILES environment variable`** —
  set `SET_FILES` to the install prefix (e.g. `/usr`).
- **`make` reports `Please reconfigure the package!`** — the generated
  `Makefile` is older than `config.pl`; re-run `./configure`.
- **Debug support** — the editor's built-in debugger needs `libmigdb`, a
  separate SourceForge project not shipped here, so it is disabled.
- **`AlCon` and `cal`** are extra tools in this repo; they are not part of the
  editor build (the editor is under `setedit/`).

## License

SETEdit is distributed under the **GNU General Public License, version 2**
(GPLv2). The full text is in [`LICENSE`](LICENSE), copied verbatim from the
original project's `setedit/copying.gpl`.

The original suite mixes a few components with slightly different terms; see
`setedit/copyrigh` for the per-component details. In short: the editor classes
and InfView are GPL-2.0-or-later, some bundled helper libraries are LGPL, and
the Robert Höhne `librhuti` sources are freely distributable.
