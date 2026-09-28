# Carbonyl on macOS — Complete Local Install Guide (PR #219 merged version)

This guide installs and runs **this repository** (Carbonyl at commit
`60e1783`, i.e. `v0.0.3` + merged PR #219) on a Mac, on both Apple Silicon
(arm64) and Intel (x86_64).

What the merged PR #219 adds on top of the `v0.0.3` release:

- Opt-in **vimium-style keyboard navigation** (`--vim` flag or
  `CARBONYL_ENV_VIM=1`): scrolling, history, in-viewport find, and link
  hints. Off by default.
- `scripts/dev-install.sh`: a helper that rebuilds only the Rust core
  (`libcarbonyl.dylib`) and hot-swaps it into an existing Carbonyl
  installation — no Chromium build required.

> **Why this guide has two "fast" paths:** Carbonyl is two pieces.
> The *runtime* is a patched Chromium headless shell (takes ~1 hour and
> 100 GB to build), and the *core* is the Rust crate in this repo
> (`libcarbonyl.dylib`, built in seconds–minutes). PR #219 only changed
> Rust code — the files that determine the runtime hash
> (`chromium/.gclient`, `chromium/patches/*`, `src/browser/*.{cc,h,gn,mojom}`)
> are unchanged since the `v0.0.3` release — so the pre-built `v0.0.3`
> runtime binaries work with this checkout as-is. You only need to
> rebuild the Rust core.

---

## 1. Prerequisites

### Common to all options

| Requirement | Notes |
| --- | --- |
| macOS (Big Sur 11 or newer recommended) | Test target per the project; no window server needed |
| Xcode Command Line Tools | `xcode-select --install` (provides `clang`, `ld`, `install_name_tool`, `strip`) |
| [Rust](https://rustup.rs) via `rustup` | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Git | Preinstalled on macOS |
| ~1 GB disk (fast paths) / ~100 GB (full build) | Option C only |

Verify the toolchain:

```console
$ xcode-select -p
/Library/Developer/CommandLineTools
$ rustc --version
$ rustup --version
```

### Full build from source (Option C) only

- 8+ CPU cores, 16 GB+ RAM (Chromium's `ninja` is memory-hungry; the
  machine will be mostly unresponsive during the build)
- ~100 GB of free disk (the `chromium/src` checkout alone is huge)
- Python 3 (some depot_tools/Chromium tooling)
- `ccache` (optional but recommended): `brew install ccache`
- Several hours of wall-clock time (fetch + build)

---

## 2. Get the source at the merged-PR commit

If you are already in a checkout of this repo at `60e1783`
(`git log --oneline -1` → `60e1783 Merge PR #219`), skip this step.

```console
$ git clone https://github.com/Zzato/carbonyl.git   # the fork where PR #219 was merged
$ cd carbonyl
$ git checkout 60e1783   # "Merge PR #219" (= the main branch at/after the merge)
```

(If your local checkout already has the merge, just use it — the steps
below only require the tree at `60e1783`.)

Note: the `chromium/depot_tools` git submodule is not checked out in a
fresh clone. The project scripts (`scripts/gclient.sh`, `scripts/gn.sh`,
`scripts/build.sh`) auto-fetch it via `git submodule update --init
--recursive` on first use, so you don't need to do anything manually.

---

## 3. Option A (recommended, ~5 minutes): pre-built runtime + rebuild the Rust core

This is the intended dev workflow for PR #219: take the official `v0.0.3`
macOS binary (whose runtime is byte-compatible with this commit) and
replace `libcarbonyl.dylib` with one built from *this* tree, giving you
the merged PR #219 code.

### 3.1 Download the v0.0.3 release binary

Pick the archive matching your architecture:

- **Apple Silicon (M1/M2/M3…):**
  <https://github.com/fathyb/carbonyl/releases/download/v0.0.3/carbonyl.macos-arm64.zip>
- **Intel:**
  <https://github.com/fathyb/carbonyl/releases/download/v0.0.3/carbonyl.macos-amd64.zip>

Check your architecture with `uname -m` (`arm64` → arm64 zip, `x86_64` → amd64 zip).

```console
$ cd ~/Downloads
$ curl -L -o carbonyl.macos-arm64.zip \
    https://github.com/fathyb/carbonyl/releases/download/v0.0.3/carbonyl.macos-arm64.zip
$ unzip carbonyl.macos-arm64.zip
$ ls ~/Downloads/carbonyl-0.0.3
carbonyl  icudtl.dat  libEGL.dylib  libGLESv2.dylib  libcarbonyl.dylib  v8_context_snapshot.arm64.bin
```

> The archive extracts to `carbonyl-0.0.3/` — exactly the default install
> directory that `scripts/dev-install.sh` expects
> (`$HOME/Downloads/carbonyl-0.0.3`). If you extracted it somewhere else,
> export `CARBONYL_INSTALL_DIR=<path>` before running the script.
>
> If you downloaded the zip through a browser (not `curl`), macOS may
> quarantine the files. If Carbonyl is killed by Gatekeeper, run:
> `xattr -dr com.apple.quarantine ~/Downloads/carbonyl-0.0.3`

### 3.2 Build the Rust core from this checkout

The build must target the same triple as the runtime binary and use
deployment target 10.13 (this is what the project's own scripts set):

```console
$ cd <path-to-carbonyl-repo>

# Apple Silicon:
$ rustup target add aarch64-apple-darwin
MACOSX_DEPLOYMENT_TARGET=10.13 cargo build --target aarch64-apple-darwin --release

# Intel:
$ rustup target add x86_64-apple-darwin
MACOSX_DEPLOYMENT_TARGET=10.13 cargo build --target x86_64-apple-darwin --release
```

This produces `build/<target-triple>/release/libcarbonyl.dylib`
(the repo's `.cargo/config.toml` sets the target dir to `build/`).

### 3.3 Install the new core into the release binary (the PR's helper)

```console
$ scripts/dev-install.sh
# Installed build/aarch64-apple-darwin/release/libcarbonyl.dylib -> /Users/you/Downloads/carbonyl-0.0.3/libcarbonyl.dylib
# Run: /Users/you/Downloads/carbonyl-0.0.3/carbonyl <url>
```

`dev-install.sh` (added in PR #219) detects the architecture of the
existing dylib, adds the right rustup target if missing, rebuilds, sets
the dylib's install name to `@executable_path/libcarbonyl.dylib` with
`install_name_tool`, and copies it over the release one.
Use `CARBONYL_INSTALL_DIR=<path>` to point at a different install
location.

### 3.4 Run it

```console
$ ~/Downloads/carbonyl-0.0.3/carbonyl https://wikipedia.org
# With the new vimium navigation from PR #219:
$ ~/Downloads/carbonyl-0.0.3/carbonyl --vim https://youtube.com
```

Optional: put it on your `PATH` for convenience:

```console
$ ln -s ~/Downloads/carbonyl-0.0.3/carbonyl /usr/local/bin/carbonyl
$ carbonyl --vim https://github.com
```

### 3.5 Verify you actually have the merged-PR core

The `v0.0.3` release core does **not** know `--vim`; only the PR #219
core does:

```console
$ ~/Downloads/carbonyl-0.0.3/carbonyl --help | grep -- --vim
        --vim                  enable vimium-style keyboard navigation
                               (h/j/k/l scroll, f link hints, / find)
```

If `--vim` shows up, you're running the merged version.

### 3.6 Re-do the swap after editing Rust code

After any change under `src/`, just re-run `scripts/dev-install.sh` —
the Chromium runtime stays untouched and the rebuild takes only a couple
of minutes. This is the documented fast loop for Rust-only changes.

---

## 4. Option B (~10 minutes): pre-built runtime from the project CDN + rebuild core

Same idea as Option A, but instead of the GitHub release zip it uses the
project's own runtime CDN (via `scripts/runtime-pull.sh`), which serves
pre-built runtimes keyed by a hash of the Chromium patches and C++ bridge
sources. This is useful if you want everything to come from this repo's
tooling (or if the release asset is unavailable).

The hash for this checkout (`60e1783`) is **`be8a22cdd571c29c`** — both
macOS runtimes are available:

- `https://carbonyl.fathy.fr/runtime/be8a22cdd571c29c/aarch64-apple-darwin.tgz` (~66 MB)
- `https://carbonyl.fathy.fr/runtime/be8a22cdd571c29c/x86_64-apple-darwin.tgz` (~74 MB)

```console
$ cd <path-to-carbonyl-repo>

# 1. Download + extract the pre-built runtime (hash is computed automatically)
$ scripts/runtime-pull.sh
# -> build/pre-built/<triple>/ containing: carbonyl, icudtl.dat,
#    libEGL.dylib, libGLESv2.dylib, v8_context_snapshot*.bin, libcarbonyl.dylib (from v0.0.3 CI)

# 2. Build the core from this tree (same commands as 3.2)
$ rustup target add aarch64-apple-darwin        # or x86_64-apple-darwin on Intel
$ MACOSX_DEPLOYMENT_TARGET=10.13 \
    cargo build --target aarch64-apple-darwin --release

# 3. Point the dylib's install name at the executable's directory
#    and swap it into the pre-built runtime
TRIPLE=aarch64-apple-darwin                     # or x86_64-apple-darwin
install_name_tool -id @executable_path/libcarbonyl.dylib \
    build/$TRIPLE/release/libcarbonyl.dylib
cp build/$TRIPLE/release/libcarbonyl.dylib build/pre-built/$TRIPLE/libcarbonyl.dylib

# 4. Run
$ build/pre-built/$TRIPLE/carbonyl --vim https://wikipedia.org
```

Verify with `build/pre-built/$TRIPLE/carbonyl --help | grep -- --vim`
as in 3.5.

> If `runtime-pull.sh` reports "Pre-built binaries not available", the
> CDN object for the current patch hash is missing — use Option A
> (release zip) or Option C (build Chromium yourself, then
> `scripts/runtime-push.sh` to republish it).

---

## 5. Option C (several hours): full build from source, Chromium included

Build the *entire* Carbonyl (patched Chromium headless shell + Rust core)
from this checkout. Do this if you modify the Chromium patches or the
C++ bridge in `src/browser/`, or just want a from-scratch build.

### 5.1 System prerequisites

```console
$ xcode-select --install            # Xcode CLT (compiler, linker)
$ brew install ccache python3 git   # ccache optional but recommended
```

Make sure you have ≥ 100 GB free on the volume where you clone the repo.

### 5.2 Fetch the Chromium source

```console
$ cd <path-to-carbonyl-repo>
$ ./scripts/gclient.sh sync
```

- First run also fetches the `depot_tools` submodule automatically.
- This downloads Chromium `src` pinned to `111.0.5511.1`
  (see `chromium/.gclient`) plus all third_party deps: tens of GB and a
  long time, depending on your connection.
- You now have `chromium/src/`.

### 5.3 Apply the Carbonyl patches

```console
$ ./scripts/patches.sh apply
```

This stashes any local Chromium edits, checks out the pinned Chromium /
Skia / WebRTC revisions, and `git am`s the patches in
`chromium/patches/{chromium,skia,webrtc}` (14 Chromium patches including
the Carbonyl library/service, software-rendering surface, and DPI
settings).

> Re-running `apply` discards any changes you made directly in
> `chromium/src`. To persist changes, use `./scripts/patches.sh save`
> from inside `chromium/src` after committing them.

### 5.4 Configure (gn)

```console
$ ./scripts/gn.sh args out/Default
```

Enter/paste the following args (this is the same set the project
documents; adjust `target_cpu` for Apple Silicon):

```gn
import("//carbonyl/src/browser/args.gn")

# Apple Silicon: uncomment. Intel: leave commented.
# target_cpu = "arm64"

# comment this to disable ccache
cc_wrapper = "env CCACHE_SLOPPINESS=time_macros ccache"

# comment this for a debug build
is_debug = false
symbol_level = 0
is_official_build = true
```

`args.gn` (from `src/browser/args.gn` in this repo) already enables the
H.264/proprietary codecs, headless ozone, software rendering setup, and
disables most unused subsystems. You can create additional targets
(`out/release`, `out/debug`, …) with the same command using different
names.

### 5.5 Build

```console
$ ./scripts/build.sh Default
```

This does, in order:

1. `cargo build --target <triple> --release` of the Rust core
   (`MACOSX_DEPLOYMENT_TARGET=10.13` is set automatically),
2. copies `libcarbonyl.dylib` into `out/Default/` and fixes its install
   name to `@executable_path/libcarbonyl.dylib`,
3. `ninja headless:headless_shell` in `out/Default/`.

Expect the Chromium compile to take ~1 hour on an 8-core machine with a
fast link (and to peg your CPU for most of it). Set
`CARBONYL_SKIP_CARGO_BUILD=1` to skip step 1 on rebuilds where only
Chromium changed.

Outputs in `out/Default/`:

- `headless_shell` — the browser executable
- `icudtl.dat`, `libEGL.dylib`, `libGLESv2.dylib`,
  `v8_context_snapshot*.bin`, `libcarbonyl.dylib`

### 5.6 Package the release layout (optional)

```console
$ ./scripts/copy-binaries.sh Default
# -> build/pre-built/<triple>/ with the executable renamed to `carbonyl`,
#    all runtime data files, libcarbonyl.dylib, and everything stripped.
```

### 5.7 Run

```console
$ ./scripts/run.sh Default https://wikipedia.org
# or the packaged binary:
$ build/pre-built/<triple>/carbonyl --vim https://youtube.com
```

---

## 6. Using Carbonyl (merged-PR features)

### 6.1 CLI flags (from `src/cli/usage.txt`)

```
Usage: carbonyl [options] [url]

Options:
    -f, --fps=<fps>    set the maximum number of frames per second (default: 60)
    -z, --zoom=<zoom>  set the zoom level in percent (default: 100)
    -b, --bitmap       render text as bitmaps
    -d, --debug        enable debug logs
        --vim          enable vimium-style keyboard navigation
                       (h/j/k/l scroll, f link hints, / find)
    -h, --help         display this help message
    -v, --version      output the version number
```

Carbonyl also forwards most native Chromium flags. Each flag has an
environment-variable equivalent: `CARBONYL_ENV_DEBUG=1`,
`CARBONYL_ENV_BITMAP=1`, `CARBONYL_ENV_SHELL_MODE=1`, and (new in
PR #219) `CARBONYL_ENV_VIM=1`.

Examples:

```console
$ carbonyl -z 80 -f 30 https://example.com
$ carbonyl --vim --debug https://youtube.com
$ CARBONYL_ENV_VIM=1 carbonyl https://wikipedia.org
```

### 6.2 Vimium navigation (new in PR #219, opt-in via `--vim`)

Enabled with `--vim` (or `CARBONYL_ENV_VIM=1`); **off by default** so all
keys reach the page. With it enabled and the URL bar unfocused:

| Key | Action |
| --- | --- |
| `j` / `k` | Scroll down / up one line |
| `d` / `u` | Scroll down / up half a page |
| `gg` / `G` | Scroll to top / bottom |
| `h` / `l` | Send Left / Right arrow to the page |
| `H` / `L` | History back / forward |
| `r` | Reload |
| `/` | Find in the current viewport (ASCII, case-insensitive); `Enter` confirms, `n`/`N` cycle matches, `Esc` clears |
| `f` | Show link-hint labels in the viewport; type the label to send a synthetic click, `Esc` cancels |
| `i` | Insert mode — keys pass through to the page (typing in inputs); `Esc` leaves |

Arrows, Enter and Tab always pass through to the page. On macOS the
built-in history shortcuts are **`Cmd+[`** (back) and **`Cmd+]`**
(forward) — Alt on other platforms. The find highlights, hint labels and
status bar are drawn as terminal cells, so they're not visible in
graphics/truecolor rendering modes, but scrolling and history keys still
work there.

### 6.3 macOS notes

- No window server required; works over SSH and in a safe-mode console.
- The URL bar is a terminal UI: type a URL and press Enter; navigation
  happens inside the terminal (alternate screen buffer).
- Fullscreen mode is not supported yet (known issue).
- If you run an Intel build on Apple Silicon under Rosetta it works, but
  build the native arm64 core/runtime for best performance.

---

## 7. Verification checklist

```console
# 1. Version (note: the PR did not bump the version, so merged-PR builds
#    still report 0.0.3 — check --help for --vim instead)
$ carbonyl --version

# 2. The merged-PR core exposes --vim
$ carbonyl --help | grep -- --vim

# 3. The core dylib is the one next to the executable
$ otool -L $(which carbonyl) | grep carbonyl
@executable_path/libcarbonyl.dylib
$ otool -D /path/to/install/dir/libcarbonyl.dylib
@executable_path/libcarbonyl.dylib

# 4. Smoke test
$ carbonyl --vim https://example.org   # browse with j/k/f/, quit with Ctrl-C
```

---

## 8. Troubleshooting

| Symptom | Fix |
| --- | --- |
| `rustup: error: toolchain 'stable' is not installed` | `rustup install stable` |
| `cargo: error: target not installed` / build fails on `-target` | `rustup target add <triple>` (see 3.2) |
| "libcarbonyl.dylib not found at …" from `dev-install.sh` | Your install dir isn't the default; `export CARBONYL_INSTALL_DIR=<dir>` containing `libcarbonyl.dylib`, or extract the release zip into `~/Downloads` |
| Architecture mismatch (`Bad CPU type` when running) | You mixed an arm64 runtime with an x86_64 core or vice versa. Rebuild the core with the same triple as the runtime (`file carbonyl` shows it) |
| dylib loads but old behavior (no `--vim`) | The swap didn't land: confirm `install_name_tool -id @executable_path/libcarbonyl.dylib` was applied to the *built* dylib and that you copied it next to the *executable* (install name is resolved relative to the executable's dir) |
| Gatekeeper kills the binary (browser-downloaded zip) | `xattr -dr com.apple.quarantine <install-dir>` |
| `gclient sync` fails mid-download (Option C) | Re-run it; it resumes. If it fails on a small dependency, check network/proxy; you may need to set `http_proxy`/`https_proxy` |
| `gn`/`ninja` errors on missing tools (Option C) | `xcode-select --install`, and `brew install ccache python3` |
| `ninja` runs out of memory (Option C) | `scripts/build.sh` doesn't forward `-j` correctly; reduce parallelism by running ninja yourself: `cd chromium/src/out/Default && ninja -j 4 headless:headless_shell` (after the cargo/copy steps, or re-run `./scripts/build.sh Default` once the OOM'd targets are done) |
| `runtime-pull.sh` says "Pre-built binaries not available" | The CDN object for this patch hash is missing for your triple — use Option A or Option C |
| Slow text / glyph artifacts | Try `-b` (`--bitmap`) to render text as bitmaps |
| High CPU while idle | This checkout contains the v0.0.3 idle-CPU fixes; if you're on an older core dylib, re-run the dev-install swap |
| Want a debug build of Chromium (Option C) | Unset `is_official_build`/set `is_debug = true` in gn args, then `./scripts/build.sh Default`; run with `-d` for Carbonyl debug logs |

### Reverting / uninstalling

The fast paths (A/B) only touch your install directory and `~/.cargo`
artifacts — just delete them:

```console
$ rm -rf ~/Downloads/carbonyl-0.0.3   # or your CARBONYL_INSTALL_DIR
$ rm -f /usr/local/bin/carbonyl       # if you symlinked it
$ rm -rf <repo>/build                 # build outputs
```

A full build (Option C) additionally occupies `chromium/src` (tens of GB):

```console
$ rm -rf <repo>/chromium/src
```

---

## 9. References

- Project readme (usage, comparisons, OS support): `../readme.md`
- Merged PR #219 (vimium navigation + `dev-install.sh`):
  commits `1c9b68c` → `3593a5c`, merged as `60e1783`
- Release binaries (v0.0.3): <https://github.com/fathyb/carbonyl/releases/tag/v0.0.3>
- Pre-built runtime CDN layout: `https://carbonyl.fathy.fr/runtime/<hash>/<triple>.tgz`,
  where `<hash>` = output of `scripts/runtime-hash.sh` (`be8a22cdd571c29c`
  at this commit) and `<triple>` is e.g. `aarch64-apple-darwin`
- Blog post: <https://fathy.fr/carbonyl>
