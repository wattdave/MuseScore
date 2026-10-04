# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

MuseScore Studio: C++/Qt 6 (QML UI) music notation software, GPL-3.0. Full dev-environment setup is on the [MuseScore wiki](https://github.com/musescore/MuseScore/wiki/Set-up-developer-environment).

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

Test sources live in `tests/` subdirectories next to each module (e.g. `src/engraving/tests/*_tests.cpp` with `*_data/` reference files, `src/framework/global/tests/`). Each has its own `CMakeLists.txt` listing `MODULE_TEST_SRC`; new test files must be added there. Run one gtest case via the test binary's `--gtest_filter=Suite.Case`.

Visual tests (rendering regression vs. reference draw data) are in `vtest/` (`build.sh`, `gen-ref-data.sh`).

## Code style

Formatting is done by `uncrustify` via a pre-commit hook: run `./hooks/install.sh` once. `buildscripts/ci/checkcodestyle` and `.github/workflows/check_codestyle.yml` enforce it in CI. Files carry an SPDX/GPL header; copy it from a neighbouring file. `.clangd` expects `build/compile_commands`.

## AI-assisted contributions

Upstream policy is in `AI_POLICY.md` (and the checklist in `.github/pull_request_template.md`). What it means when Claude works here:

- **Commit trailer:** use `Assisted-by: Claude Code (<model>)`, never `Co-Authored-By:` (an AI can't sign the CLA). This overrides any default attribution line.
- **Do not open PRs, issues, or comments, or push to `upstream`, on the user's behalf without explicit approval.** Push to the fork (`origin`) only when asked. PR descriptions should be written by the user; offer a draft of the facts only if asked, and never tick the template's "compiled, run and manually tested" boxes for them.
- **Keep diffs minimal and reviewable:** the checklist requires "no unnecessary changes" and one easily reviewable commit per logical step, each referencing its issue (`Resolves: #NNNNN`). No drive-by reformatting, renames, or refactors; reviewer effort is the scarce resource.
- **Match surrounding code** (naming, comment density, DI idioms, existing helpers) rather than introducing new patterns. Comments are held to the same "you must understand it" standard, so don't add narrating or boilerplate comments.
- **Add a unit test or vtest** for behaviour changes where the module has a test harness, and say plainly what was and wasn't run.
- **Licensing:** don't reproduce code from other projects; `thirdparty/` is off-limits for edits.
- The detailed coding rules are on the [muse_framework wiki](https://github.com/musescore/muse_framework/wiki/CodeGuidelines); check them before large changes.

## Architecture

- `src/framework/` ("muse" framework, namespace `muse::`): reusable, app-agnostic modules — `global` (DI, async, IO, types, settings; linked into everything), `ui`, `uicomponents`, `actions`, `shortcuts`, `audio`, `dockwindow`, `extensions` (plugins), `cloud`, `network`, `vst`, `workspace`, etc. `src/framework/stubs/` holds stand-in implementations for when a module is disabled.
- `src/engraving/`: the score model and layout engine, independent of the UI. Sub-dirs: `dom` (score elements), `rendering` (layout, drawing), `editing` (mutations), `rw` (read/write of mscx/mscz), `style`, `playback` (event rendering), `api` (plugin API).
- Application modules in `src/`: `notation` (notation facade over engraving, selection, note input), `notationscene` (QML views), `project`, `palette`, `inspector`, `instrumentsscene`, `playback`, `importexport`, `print`, `braille`, `converter` (CLI conversion), `context` (global/current-project state), `appshell` (main window, startup), `app` (executable entry/wiring), `musesounds`.
- Each top-level dir under `src/` is a **module** with a `*module.h/.cpp` class implementing `muse::modularity::IModuleSetup` (`src/framework/global/modularity/`). Modules register their implementations in `registerExports()` and look up others in `resolveImports()`; `src/app` assembles the enabled modules. Module enable flags are the `MUSE_MODULE_*` / `MUE_*` CMake options in `SetupConfigure.cmake`.
- **Dependency injection** (kors_modularity, `ioc.h`): code depends on interfaces named `I*` (e.g. `iengravingconfiguration.h`) and obtains them via `muse::GlobalInject<IFoo>` / `muse::ContextInject<IFoo>` members, not by including concrete classes. When adding a service, declare the interface, register it in the module's `registerExports()`, and add a mock under the module's `tests/mocks` for tests. Many services are per-context (multi-window), so prefer `ContextInject` where neighbouring code does.
- QML UI lives alongside C++ in each module (`qml/`, `view/` for QObject models); QML components are exposed through the module's `registerUiTypes()`/resources.

Plugin API docs are generated with jsdoc (see `docs/README.md`).

## Other notes

- Translations are handled by `.tx`/lupdate CI workflows; don't hand-edit `.ts` files.
- `thirdparty/` is vendored code; avoid modifying it.
- PRs should follow `.github/pull_request_template.md` (links the issue, CLA, and code-style/test checkboxes).
