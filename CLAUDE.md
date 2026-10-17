# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of jdtsmith/emacs-mac (the "Mac port" of GNU Emacs, originally by
Mitsuharu Yamamoto). The `emacs-mac-31` branch is `upstream/emacs-mac-31`
plus local commits replayed on top; sync by rebasing onto upstream. Remotes:
`origin` = liaowang11/emacs-mac, `upstream` = jdtsmith/emacs-mac. There is no
GNU remote; fetch GNU refs from `https://github.com/emacs-mirror/emacs.git`
(savannah times out from this network).

The Mac port is a separate window system (`window-system` = `mac`,
`HAVE_MACGUI`), not the NS/Cocoa port. The `ns*` files are the official NS
port and are normally not what you want to touch. `README-mac` describes
user-facing features; `devel_update_notes.org` records what broke when GNU
releases were merged in (useful for the next merge).

## Build

`./autogen.sh` writes `transfer.fsckObjects` and `diff.*.xfuncname` into the
repo's git config and installs Emacs's own git hooks into `.git/hooks`
(shared by all worktrees). To avoid that, and to keep build products out of
the checkout, build from a copy:

```sh
rsync -a --exclude .git ./ /tmp/emacs-mac-build/
cd /tmp/emacs-mac-build
./autogen.sh autoconf            # autoconf from /opt/homebrew/bin; automake not needed
./configure --with-gnutls=ifavailable --with-xml2=ifavailable \
  --with-tree-sitter=ifavailable --with-sqlite3=ifavailable --with-json=ifavailable
make -j16
```

Plain `./configure` aborts on missing gnutls. A build without native-comp
or image libraries takes a few minutes; the expected warnings are only
`ranlib: ... has no symbols` from gnulib stubs. The app bundle lands in
`mac/Emacs.app`; the binary in `src/emacs`.

## Tests

ERT tests live under `test/`, mirroring the source tree
(`test/src/fns-tests.el` tests `src/fns.c`, `test/lisp/foo-tests.el` tests
`lisp/foo.el`). From a configured build tree:

```sh
make -C test src/fns-tests                       # one test file
make -C test src/fns-tests SELECTOR='"name-regexp"'   # subset
make check                                       # default selector skips :expensive-test, :unstable
```

Mac GUI behavior cannot be tested in `--batch`. Run a script in a real GUI
instance without stealing focus, writing results to a file and ending with
`kill-emacs`:

```sh
open -g -n -a /tmp/emacs-mac-build/mac/Emacs.app --args -Q -l /tmp/test.el
```

`test/src/mac-transparent-titlebar-tab-bar-tests.el` is such a script, not a
batch ERT file.

## Architecture: two threads

The central fact of this port, and the source of most of its bugs, is that
Emacs runs on two threads (`src/macappkit.m`):

- `main` creates a **Lisp thread** (`mac_start_lisp_main`) that runs the
  whole Lisp interpreter, redisplay and GC, then turns the process's main
  thread into the **GUI thread**, which runs `mac_gui_loop`.
- AppKit work must happen on the GUI thread. Lisp-side C code hands it over
  synchronously with `mac_within_gui (^{ ... })`, using a block queue and a
  pair of dispatch semaphores. The reverse direction, GUI → Lisp, is
  `mac_within_lisp` and its `_deferred*` variants; it only works when the
  Lisp thread is waiting in a known place (e.g. `mac_select`, which replaces
  `select` so the Lisp thread can serve GUI requests while idle).
- Synchronous AppKit callbacks on the GUI thread (accessibility, input
  method, `respondsToSelector:`, drawing) sometimes read Lisp data directly.
  That is only safe when the Lisp thread is not mutating it (not in GC, not
  in `inhibit-quit`/select emulation). Code that does this has explicit
  guards; any new GUI-thread access to Lisp objects needs the same. A
  SIGSEGV on the GUI thread inside such a callback tends to become a
  permanent hang rather than a crash, because the fatal-signal handler
  forwards the signal to the Lisp thread, which then blocks on the GUI
  thread.

Layering of the Mac-specific C/Obj-C sources:

| File | Role |
|------|------|
| `src/macappkit.m` / `.h` | All Objective-C/AppKit code: thread protocol, NSApplication/window/view subclasses, frame controllers, accessibility, input method, tab groups, toolbar, SVG/web views |
| `src/macterm.c` / `.h` | Terminal back end (`redisplay_interface`, `read_socket`, drawing glue); plain C calling into macappkit |
| `src/macfns.c` | Frame parameters and `x-*`/`mac-*` Lisp primitives |
| `src/macmenu.c`, `src/macselect.c` | Menus; selections, drag and drop, Apple events |
| `src/macfont.m` | Core Text font back end |
| `src/mac.c` | Non-GUI Mac utilities (CF ↔ Lisp conversion, AppleScript, etc.) |
| `lisp/term/mac-win.el` | Lisp side of the window system: key/event maps (`mac-apple-event-map`), mouse/gesture handling, startup |
| `mac/` | `Emacs.app` bundle template and its `Makefile.in` |

Mac-port code inside shared files (`xdisp.c`, `frame.c`, `image.c`,
`xfaces.c`, Makefiles, `configure.ac`) is guarded by `HAVE_MACGUI`,
`(featurep 'mac-win)` or `(eq window-system 'mac)`. These guards are where
GNU merges conflict.

## Conventions

- Commit subjects follow the recent local style: `file-or-area: summary`
  in lowercase, e.g. `macappkit: bound SVG web view waits`, without issue
  references (rebases re-push them and spam cross-references).
- C follows GNU style (2-space indent, space before the paren in calls,
  tabs for 8 columns), as in the surrounding code.
- `accessibility-coordination-plan.md` (untracked) is a design note for
  replacing the GUI-thread "peek" accessors with GUI ↔ Lisp coordination.
