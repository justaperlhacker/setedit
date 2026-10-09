# SETEdit Revival — Plan (status: BUILDS + RUNS + PUBLISHED, 2026-10-09)

Published at **https://github.com/justaperlhacker/setedit**.

Revive the canonical SETEdit suite so it compiles and runs on a modern x86-64
Linux toolchain, mirroring what was done for RHIDE. This is the *editor*
project itself (standalone `setedit`/`infview`), not just the `libset` library
RHIDE borrows.

## Current state (already validated on this host)

- Source: `git clone https://git.code.sf.net/p/setedit/code setedit-code`
  (SourceForge canonical repo, HEAD `d8231bc`, SETEdit **v0.5.8**).
- Toolchain: gcc/g++ **16.2.1**, GNU make **4.4.1**, Perl, binutils.
- Turbo Vision: the modern fork `justaperlhacker/tvision` @ `modern-gcc`
  (v2.2.3), built static (`./configure --without-dynamic && make`).
- Result: `makes/editor.exe` (3.2 MB) and `makes/infview.exe` (1.08 MB) build
  clean with `make -j$(nproc)` and the editor runs as a native **X11** window
  (`WM_CLASS = tvapp.SETEdit`, title `SETEdit v0.5.8 - No project loaded`).
- Only two small `setedit/config.pl` changes were needed (below).

> Note: the SourceForge `setedit/code` repo is a **different history** from the
> GitHub mirror `set-soft/setedit` (the tree RHIDE currently consumes). The SF
> repo is the canonical one; both report v0.5.8.

## Goal

1. Fork/revive the canonical tree under `github.com/justaperlhacker`.
2. Build the full editor (`setedit` + `infview`) with current build tools.
3. Run under X11 (and terminal), like RHIDE.
4. Ship the same working artifacts RHIDE has: `README.md`, `REVIVAL-PLAN.md`,
   `setup-new-host.sh.txt`.

## Prerequisites

- `gcc`, `g++`, `make`, `perl`, `git`.
- Dev libs: ncurses, gpm, X11, Xmu, zlib, bzip2, pcre.
  On Arch/CachyOS: `sudo pacman -S --needed base-devel ncurses gpm libx11 libxmu zlib bzip2 pcre perl git`.
  (The tree also ships fallback copies of zlib/bzip2/pcre and uses the system
  ones when detected, which is the default here.)
- Turbo Vision static fork built first (see RHIDE's README step 1).

## Build (from scratch)

```sh
# 0. Turbo Vision (static), modern-gcc fork -- see rhide README
cd ~/Projects/tvision/tvision
./configure --without-dynamic && make -j"$(nproc)"
ln -sf "$PWD/rhtv-config" ~/.local/bin/rhtv-config

# 1. canonical SETEdit tree
cd ~/Projects
git clone https://git.code.sf.net/p/setedit/code setedit-code

# 2. full editor
cd setedit-code/setedit
./configure \
  --tv-include="$HOME/Projects/tvision/tvision/include" \
  --tv-lib="$HOME/Projects/tvision/tvision/makes"
make -j"$(nproc)"
# -> makes/editor.exe, makes/infview.exe
```

Install (optional): `make install` (prefix `/usr`) — the Linux install path
runs `makes/linux/compress.pl` and installs the editor as `setedit`.

RHIDE-compatible library (optional, unchanged interface):
`./configure --libset --no-infview --tv-include=... --tv-lib=... &&
make needed && make libset`.

## Fixes required (both in `setedit/config.pl`)

1. **Missing TV runtime libs on the link line.**
   On UNIX the script asked `rhtv-config --dlibs` for the TV libraries unless
   `--static` was given. For a static-only TV build (the fork) `--dlibs`
   returns only `-lrhtv`, omitting ncurses/gpm/X11/Xmu, so `editor.exe` failed
   to link (`undefined reference to stdscr/wgetch/X...`). `--slibs` returns
   the full list with the `-Wl,-dn -lrhtv -Wl,-dy ...` grouping. Fix: always
   use `TVConfigOption('slibs')` in the UNIX branch (do **not** pass `-static`,
   which forces a full static link and fails against dynamic system libs).

2. **`make -j` race building `infview`.**
   `all: Makefile editor infview`; `editor` depends on `needed` but `infview`
   did not, so a parallel link could start before `libmpegsnd.a` existed. Fix:
   generate `infview: needed`.

Verified: after those two edits, a fresh `make -j$(nproc)` (with the mp3 lib
deleted first) builds both binaries with exit 0.

## Verification performed

- `./configure` auto-detects Turbo Vision 2.2.3, zlib 1.3.1, bzip2 1.0.8,
  PCRE >= 2.0.6, dl; exits "Successful configuration!".
- `make -j$(nproc)` → `editor.exe` + `infview.exe`, no errors.
- `env DISPLAY=:0 SET_FILES=/usr ./makes/editor.exe` opens an X11 window
  (`xdotool`/`wmctrl` show class `tvapp.SETEdit`). `SET_FILES` is required or
  the editor aborts with "Wrong installation! You must define SET_FILES".
  Killing it with SIGTERM makes it dump a panic file under `~/.setedit/` (by
  design, harmless).
- `infview.exe` links and is produced alongside.
- Warnings only: `-Wold-style-definition` in the bundled `fstrcmp.c`,
  `-Wregister` / `-Waggressive-loop-optimizations` in the bundled mp3 decoder.
  No `-Werror`, so they are cosmetic.

## GitHub project — DONE

Published: **https://github.com/justaperlhacker/setedit** (public, default
branch `master`, 15 upstream tags pushed, revival commit `7748066`).

**Account mechanics (important):** the `gh` CLI and the `GITHUB_TOKEN` in
`~/.config/opencode/.env` on this host both belong to **`satrac`**, which has
only read access to `justaperlhacker/*` and therefore cannot create
repositories there (`gh repo create justaperlhacker/setedit` →
*"satrac cannot create a repository for justaperlhacker"*). The SSH key
`~/.ssh/id_ed25519_netmancer` (`Host github-justaperlhacker`) authenticates as
**justaperlhacker** — but an SSH alias only carries `git fetch/push`, it is not
an API credential. The empty repo was therefore created by the
`justaperlhacker` account (web UI); the push needs no token.

Remotes (already configured):
```sh
origin   git@github-justaperlhacker:justaperlhacker/setedit.git
upstream https://git.code.sf.net/p/setedit/code
```

Optional consistency (declined for now): point RHIDE's `setup-new-host.sh.txt`
/ README at `justaperlhacker/setedit` instead of `set-soft/setedit`. RHIDE only
needs `--libset`, which this tree still provides unchanged.

## Repo layout after revival

- root additions: `README.md`, `BUILD.md`, `REVIVAL-PLAN.md`,
  `setup-new-host.sh.txt`, `.gitignore`.
- upstream tree unchanged except `setedit/config.pl`: `AlCon/`, `cal/`,
  `setedit/`.

## Open decisions

- Chase `libmigdb` for the editor's built-in debugger? (separate SF project;
  not shipped here, so debug features are disabled — RHIDE has its own GDB).
- Build/verify `AlCon` (Allegro conio emulation) and `cal`? Currently out of
  scope (DOS/legacy).

## Not yet done

- `make install` / `.deb` / `.rpm` packaging not exercised (prefix install
  path is `makes/linux/compress.pl`).
- No security/correctness review of the 1996-2017 C/C++ sources.
- Terminal (non-X11) run not re-verified here (X11 verified).
