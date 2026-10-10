# SETEdit Enhancement Roadmap

Ideas for developing the revived SETEdit beyond "it builds and runs". Nothing
here is committed to; it's a prioritized backlog.

Context at the time of writing:

- `setedit` builds with gcc 16 (X11 + terminal drivers via the
  `justaperlhacker/tvision` `modern-gcc` fork) and runs as `editor.exe` /
  `infview.exe`.
- The same tree provides `libset.a`, which `justaperlhacker/rhide` now links.
- License: GPLv2 (mixed component terms; see `setedit/copyrigh`).
- Known gaps: no built-in debugger (`libmigdb` not shipped), byte/codepage
  text model, requires `SET_FILES`, no CI/packaging.

Legend: **[S]** setedit, **[TV]** belongs in the tvision fork.

---

## Quick wins (low risk, high value)

- [ ] **[S] CI**: GitHub Actions that builds setedit against the tvision
  `modern-gcc` fork on every push; gcc + clang matrix.
- [ ] **[S] Remove the `SET_FILES` requirement**: fall back to a compile-time
  install prefix instead of aborting with "Wrong installation!".
- [ ] **[S] XDG config**: move state to `~/.config/setedit` (with migration
  from `~/.setedit`); tidier crash-recovery files than `erlXXXXXX`.
- [ ] **[S] Packaging**: the tree has `debian/` and `redhat/`; add an AUR
  `PKGBUILD` and an AppImage, and exercise `make install` in CI.
- [ ] **[TV] Fix the root cause**: make `rhtv-config --dlibs` return the TV
  dependencies when only a static `librhtv.a` exists, so setedit's
  `config.pl` `--slibs` workaround is no longer needed.
- [ ] **[S] Warning hygiene**: clear the C++17 `register` and
  `-Wold-style-definition` warnings, enable `-Wall -Wextra`, and add
  `cppcheck`/`clang-tidy` plus an **ASan/UBSan** CI job.
- [x] **[S] Build: config changes force a rebuild** — `config.pl` now drops
  stale object files when `include/configed.h` changes, so a new `--prefix` is
  actually compiled in (previously the installed binary kept the old
  `CONFIG_PREFIX` and demanded `SET_FILES`).

## Medium

- [ ] **[S] PCRE2** instead of the bundled ancient PCRE; prefer system
  zlib/bzip2 by default.
- [ ] **[S] Modern syntax highlighting**: Rust, Go, TypeScript, Zig, TOML,
  YAML, Raku (reuse the `tree-sitter-raku` work) in `syntaxhl.shl`.
- [ ] **[S] Color themes**: reuse the color-theme system recently added to
  RHIDE.
- [ ] **[S] Git integration**: status / diff / blame inside the editor.
- [ ] **[S] Docs**: rebuild info/man pages under makeinfo 7, add a `.desktop`
  file and icons.

## Large projects

- [ ] **[S] Debugger**: wire in `libmigdb` (separate SourceForge project, not
  shipped) or drive the system GDB over MI. The headline missing feature.
- [ ] **[S] UTF-8 / Unicode-native editing**: SETEdit is still byte and
  codepage oriented; deep but high impact.
- [ ] **[S] LSP client + tree-sitter highlighting** (leverage `tree-sitter-raku`
  / `zed-raku`).
- [ ] **[TV] Wayland-native Turbo Vision driver** (X11 currently works through
  XWayland).
- [ ] **[TV] TTF/OTF font support**: a FreeType backend for the TV X11 driver
  (`classes/x11/x11src.cc`) that rasterizes monospace outlines into the
  driver's per-glyph `XImage`. Start with `FT_RENDER_MODE_MONO` to reuse the
  existing `XPutImage` path, then antialiasing via 8-bit images / XRender.
  Needs codepage → Unicode glyph mapping (ties into the UTF-8 item) plus
  `FontFile` / `FontSize` / bold+italic options. Terminal and console drivers
  can't use outlines, so this only applies to graphical drivers.
- [ ] **[S] 64-bit/portability audit + fuzzing** of the loaders/parsers
  (`loadshl`, `tags`, macros) — old C code, good ASan/fuzzer targets.

---

## Suggested first batch

1. CI + build matrix (quick, prevents regressions).
2. Install-prefix / `SET_FILES` fix + packaging.
3. Warning cleanup + sanitizer job.
4. PCRE2 + modern syntax files + themes.

## Cross-repo note

Items marked **[TV]** are better fixed in `justaperlhacker/tvision` (the
`modern-gcc` branch) so RHIDE benefits too; keep setedit's `config.pl`
workarounds only until then.
