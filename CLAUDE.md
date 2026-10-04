# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

MuseScore Studio: C++/Qt 6 (QML UI) music notation software, GPL-3.0. Full dev-environment setup is on the [MuseScore wiki](https://github.com/musescore/MuseScore/wiki/Set-up-developer-environment).

## Getting sources

The framework and prebuilt-dependency recipes are **git submodules**: `muse/` ([muse_framework](https://github.com/musescore/muse_framework)) and `muse_deps/` (dependency recipes/lockfile). Clone with `--recurse-submodules`, or run `git submodule update --init` after checkout/pull; without them nothing builds and `muse/framework/...` paths below are empty. Framework changes are made in the `muse/` repo (its own fork/branch/PR); the `wf` script automates the fork/branch/sync dance for both repos (`wf -t make-branch -b <name>`, needs `mf_fork_url.local`).

## macOS environment (verified on Apple Silicon)

The wiki's setup pages are partly stale; what actually worked:

- `brew install cmake git libsndfile pkg-config ninja ccache uncrustify`. CMake must be >= 3.31 on macOS.
- **Full Xcode is required** (not just the Command Line Tools), despite what the wiki says: the build has Swift/Quick Look targets. If Xcode isn't the active developer dir and you can't `sudo xcode-select -s`, set `DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`.
- **Qt: match the release.** `main` and CI use 6.10.2; the online installer may only offer newer patch releases. `main` builds and starts on 6.10.3, but the 4.7.5 tag fails at startup on 6.10.3 (`LayoutPanel.qml: "__currentRowItem" is a read-only property`). Install an exact version without a Qt account via `aqtinstall` in a venv: `aqt install-qt mac desktop 6.10.2 clang_64 -m qt5compat qtnetworkauth qtshadertools --outputdir ~/Qt` (run it outside the repo; it writes `aqtinstall.log` to the cwd).
- `build.cmake` finds Qt/Xcode via a git-ignored `build_overrides.cmake` in the repo root: `set(ENV{QTDIR} "$ENV{HOME}/Qt/6.10.2/macos")`, `set(ENV{DEVELOPER_DIR} ...)`, `list(APPEND CONFIGURE_ARGS -G Ninja)`. `ninja_build.sh` does **not** read it; export `QTDIR`, `CMAKE_PREFIX_PATH`, `PATH` (Qt `bin`) and `DEVELOPER_DIR` instead.
- `ninja_build.sh` defaults to 4 jobs on macOS (no `nproc`); pass `-j $(sysctl -n hw.ncpu)`.
- Output dirs: `build.cmake` uses `builds/<Platform>-Qt<ver>-<qt compiler>-<generator>-<type>/` (so changing Qt version gives a fresh dir; app at `install/mscore.app`), `ninja_build.sh` uses `build.debug/` / `build.release/`.
- A cold build takes roughly 3-5 minutes with ccache warm on a 12-core machine; the first one downloads prebuilt dependencies from `muse_deps` releases.

## Build & run

Two entry points, both create an out-of-tree build dir in the repo root:

```bash
# CMake script (cross-platform; args: clean, configure, build, install, run)
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Debug          # default type is RelWithDebInfo
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Debug run      # args after "run" go to the app
cmake -P build.cmake -DCMAKE_BUILD_TYPE=Debug clean    # wipe build dir, then retry

# Ninja wrapper (what CI uses); targets: release, debug, relwithdebinfo, install, installdebug, ...
bash ./ninja_build.sh -t debug                         # output in build.debug/
```

`ninja_build.sh` is configured through env vars (`MUSESCORE_BUILD_CONFIGURATION`, `MUSESCORE_BUILD_UNIT_TESTS`, `MUSESCORE_COMPILE_USE_UNITY`, `MUSESCORE_DOWNLOAD_SOUNDFONT`, ...). `MUSESCORE_BUILD_CONFIGURATION` selects a preset in `SetupConfigure.cmake` (`app`, `utest`, and reduced configs such as the WASM / no-Qt / backend builds that switch off many `MUSE_MODULE_*` options).

## Tests

Unit tests use GoogleTest and are only built when enabled. Mirror CI (`buildscripts/ci/linux/build_utest.sh` + `runutests.sh`):

```bash
MUSESCORE_BUILD_CONFIGURATION=utest MUSESCORE_BUILD_UNIT_TESTS=ON \
MUSESCORE_DOWNLOAD_SOUNDFONT=OFF bash ./ninja_build.sh -t debug

cd build.debug
export QT_QPA_PLATFORM=minimal:enable_fonts           # headless
ctest -V                                              # all tests
ctest -R <regex> -V                                   # subset
```

The wiki's `-DMUE_BUILD_UNIT_TESTS=ON` is obsolete; tests are enabled by `MUSESCORE_BUILD_CONFIGURATION=utest` (sets `MUSE_ENABLE_UNIT_TESTS`, also turns on ASan and unity builds).

Test sources live in `tests/` subdirectories next to each module (e.g. `src/engraving/tests/*_tests.cpp` with `*_data/` reference files, `muse/framework/global/tests/`). Each has its own `CMakeLists.txt` listing `MODULE_TEST_SRC`; new test files must be added there. Run one gtest case via the test binary's `--gtest_filter=Suite.Case`.

Known test results on macOS/arm64 (AppleClang/libc++, Qt 6.10.2 and 6.10.3): 32/33 suites pass. `engraving_tests` fails 14 cases (`MidiRenderer_Tests.vibrato` and 13 `MidiRendererBend_Tests.*`) that pass on Linux CI. Cause: the helpers `checkEventInterval`/`checkPitchBend` in `src/engraving/tests/midi/midirenderer_tests.cpp` call `std::multimap::find(tick)` and assume it returns the first of several events at the same tick; libc++ returns a different duplicate (libstdc++ returns the first), so a note-on and a pitch bend at the same tick get swapped. It is a test-portability issue, not a renderer bug; the fix is `lower_bound` in those helpers (fork branch `fix-midirenderer-tests-multimap-find`, issue wattdave/MuseScore#2, not upstreamed). Upstream `main`'s scheduled CI is itself red (e.g. `Engraving_CopyPasteTests.copypasteparts` on 2026-10-01), so check CI before assuming a failure is yours.

Visual tests (rendering regression vs. reference draw data) are in `vtest/` (`build.sh`, `gen-ref-data.sh`).

## Code style

Formatting is done by `uncrustify` (config: `muse/tools/codestyle/uncrustify_muse.cfg`, in the submodule; check a file with `uncrustify -q -c <cfg> -f <file> | diff - <file>`) via a pre-commit hook: run `./hooks/install.sh` once. CI (`.github/workflows/check_codestyle.yml`, using `muse/buildscripts/ci/checkcodestyle/`) fails PRs with violations. Don't fight the formatter; if its output is wrong, fix the config in a separate PR. `.clangd` expects `build/compile_commands`.

Rules from the [CodeGuidelines wiki](https://github.com/musescore/muse_framework/wiki/CodeGuidelines) that the formatter does not enforce (the wiki is partly stale: it says namespaces are `mu::` and cites an old config path; trust the code):

- **Naming:** `camelCase` variables/functions, `PascalCase` types, acronyms camel-cased (`XmlReader`), class data members `m_name`, public struct (POD) members unprefixed, constants `UPPER_SNAKE`. Avoid abbreviations and meaningless names; single letters only for obvious counters/temporaries.
- **Namespaces:** lowercase. The app uses `mu::<module>`, the framework uses `muse::<module>`.
- **Variables:** one per line, declared where first needed, always initialized, `nullptr` not `0`. `Type* p` / `const Type& r` (no space after `*`/`&`).
- **Braces:** attached, except function bodies and class declarations (brace on its own line). Always brace one-line `if`/`for` bodies. No `else` after `return`. Constructor initializer lists start with `:` on a new line.
- **Parentheses/line breaks:** parenthesize mixed `&&`/`||` and bitwise expressions. Lines < 120 chars; wrapped commas trail, wrapped operators lead.
- **Includes:** only what you use; prefer forward declarations in headers. Order, with a blank line between groups and alphabetical within: own header, system, Qt, framework, IOC (`ioc.h`, framework, other modules, current module), current module, current class.
- **Headers:** license header first; `#pragma once` is the norm (most files; `#ifndef` guards exist too, match the neighbours). Declarations only (no singleton implementations in headers). Class layout: ctor/dtor, public, public slots, signals, protected, private slots, private methods, private members.
- **Casts:** never C-style. `static_cast` for fundamental/enum/alias types and certain up/down-casts; for `EngravingObject` use `isChord()`/`toChord(e)` rather than `static_cast`; `qobject_cast` for `QObject`s; `dynamic_cast` only otherwise (RTTI is slow); `reinterpret_cast` essentially never. Use `auto` when the type is repeated on the line (e.g. after a cast) or very long.
- **QML:** same style; property order is `id`, anchors/Layout, regular properties, `navigation.*`, then signal handlers, each group separated by a blank line. Opening brace always on the same line. Private properties go in a nested `QObject { id: prv }`.
- **Bash:** Google shell style variant (`set -euo pipefail`, `local` vars, messages to stderr). **Python:** `PascalCase` classes, `snake_case` everything else.
- Style does not apply to third-party code or to files already in a different style; keep a file's existing style.

## AI-assisted contributions

Upstream policy is in `AI_POLICY.md` (and the checklist in `.github/pull_request_template.md`). What it means when Claude works here:

- **Commit trailer:** use `Assisted-by: Claude Code (<model>)`, never `Co-Authored-By:` (an AI can't sign the CLA). This overrides any default attribution line.
- **Do not open PRs, issues, or comments, or push to `upstream`, on the user's behalf without explicit approval.** Push to the fork (`origin`) only when asked. PR descriptions should be written by the user; offer a draft of the facts only if asked, and never tick the template's "compiled, run and manually tested" boxes for them.
- **Keep diffs minimal and reviewable:** the checklist requires "no unnecessary changes" and one easily reviewable commit per logical step, each referencing its issue (`Resolves: #NNNNN`). No drive-by reformatting, renames, or refactors; reviewer effort is the scarce resource.
- **Match surrounding code** (naming, comment density, DI idioms, existing helpers) rather than introducing new patterns. Comments are held to the same "you must understand it" standard, so don't add narrating or boilerplate comments.
- **Add a unit test or vtest** for behaviour changes where the module has a test harness, and say plainly what was and wasn't run.
- **Licensing:** don't reproduce code from other projects.
- The `muse/` submodule has its own `AI_POLICY.md` and PR flow; the same rules apply to framework changes.
- Follow the style rules above; generated code that merely compiles is not enough, and style churn counts as an "unnecessary change".

## Architecture

- `muse/framework/` (submodule; "muse" framework, namespace `muse::`): reusable, app-agnostic modules — `global` (DI, async, IO, types, settings; linked into everything), `ui`, `uicomponents`, `actions`, `shortcuts`, `audio`, `dockwindow`, `extensions` (plugins), `cloud`, `network`, `vst`, `workspace`, etc. `muse/framework/stubs/` holds stand-in implementations for when a module is disabled.
- `src/engraving/`: the score model and layout engine, independent of the UI. Sub-dirs: `dom` (score elements), `rendering` (layout, drawing), `editing` (mutations), `rw` (read/write of mscx/mscz), `style`, `playback` (event rendering), `api` (plugin API).
- Application modules in `src/`: `notation` (notation facade over engraving, selection, note input), `notationscene` (QML views), `project`, `palette`, `propertiespanel`, `instrumentsscene`, `playback`, `importexport`, `print`, `braille`, `converter` (CLI conversion), `context` (global/current-project state), `appshell` (main window, startup), `app` (executable entry/wiring), `musesounds`.
- Each top-level dir under `src/` is a **module** with a `*module.h/.cpp` class implementing `muse::modularity::IModuleSetup` (`muse/framework/global/modularity/`). Modules register their implementations in `registerExports()` and look up others in `resolveImports()`; `src/app` assembles the enabled modules. Module enable flags are the `MUSE_MODULE_*` / `MUE_*` CMake options in `SetupConfigure.cmake`.
- **Dependency injection** (kors_modularity, `ioc.h`): code depends on interfaces named `I*` (e.g. `iengravingconfiguration.h`) and obtains them via `muse::GlobalInject<IFoo>` / `muse::ContextInject<IFoo>` members, not by including concrete classes. When adding a service, declare the interface, register it in the module's `registerExports()`, and add a mock under the module's `tests/mocks` for tests. Many services are per-context (multi-window), so prefer `ContextInject` where neighbouring code does.
- QML UI lives alongside C++ in each module (`qml/`, `view/` for QObject models); QML components are exposed through the module's `registerUiTypes()`/resources.

Plugin API docs are generated with jsdoc (see `docs/README.md`).

## Running a dev build for debugging

- A dev build keeps its user data separate from the installed app: `~/Documents/MuseScoreStudio5Development/` (+ `~/Library/Application Support/MuseScore/MuseScoreStudio5Development/`) for `main`, `MuseScore4Development` for the 4.7.x line. Plugins go in `<Documents dir>/Plugins/<name>/<name>.qml`.
- Plugin enabled state is `<Application Support dir>/extensions/config.json`, and its format differs by version: `main` uses `[{"uri":"extension://plugin/v1/<name>/<name>.qml","enabled":true}]`; 4.7.x uses `[{"uri":"musescore://extensions/v1/<name>/<name>.qml","actions":[{"code":"main","exec_point":"manually"}]}]`. Write it while the app is not running.
- To get a stack trace, launch under `lldb -b -o run -k "thread backtrace all" -k "frame info" -k "kill" -o quit -- <mscore> <score>` (`-k` runs on crash; `-o` commands after `run` do not). Don't SIGTERM an app that lldb is attached to; use `kill -9` on both, and expect a "previous session quit unexpectedly" prompt and occasionally a CoreMIDI assert on the next launch (retry).
- The app ignores synthetic mouse clicks inside its QML windows. Menu-bar items can be clicked with `osascript` (`click menu item ... of menu 1 of menu bar item ...`, the plugin menu is "Extensions" on `main`, "Plugins" on 4.7.x). For dialog buttons, set `AXFocused` on the button via System Events and send `key code 49` (Space); `AXPress` and coordinate clicks do nothing. `screencapture -x` + Read lets you see the UI.

## Other notes

- Translations are handled by `.tx`/lupdate CI workflows; don't hand-edit `.ts` files.
- `muse_deps/` pins prebuilt third-party dependencies (recipes + `prebuilt.lock`); avoid editing it casually.
- PRs should follow `.github/pull_request_template.md` (links the issue, CLA, and code-style/test checkboxes).
