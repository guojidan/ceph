# Repository Guidelines

## Communication Rules
- Respond to the user in Chinese; use English in all other contexts.

## Project Structure & Module Organization
- `src/` contains the core C++ daemons (OSD, MON, MDS, RGW) and shared libraries; place new services under the matching subsystem tree.
- `src/test/` hosts unit and functional targets; `qa/` carries Teuthology suites and regression scenarios.
- `doc/` holds Sphinx sources, while `admin/` and `cmake/` supply build scripts, packaging helpers, and CI logic—edit alongside build changes.
- Top-level `build/` is disposable CMake output; recreate it locally and never commit it.

## Build, Test, and Development Commands
- `./install-deps.sh` installs platform prerequisites; rerun after system upgrades or when adding dependencies.
- `./do_cmake.sh -DCMAKE_BUILD_TYPE=RelWithDebInfo` configures a debug-friendly build tree in `build/`; pass extra `ARGS="..."` to surface feature toggles (e.g., `-DWITH_RADOSGW=OFF`).
- `cmake --build build -j$(nproc)` (or `cd build && make`) compiles default targets; use `--target <name>` for focused rebuilds such as `vstart`.
- `./run-make-check.sh` orchestrates a full developer validation, combining compilation, unit suites, and selected integration smoke tests.

## Coding Style & Naming Conventions
- Follow `CodingStyle`: C adopts Linux kernel rules; C++ mirrors Google style with Ceph adjustments (`m_` members, `SOME_CONST` constants, snake_case functions); indent with two spaces.
- Python modules are PEP 8 compliant; JavaScript/TypeScript should pass the Angular + Prettier toolchain noted in `CodingStyle`.
- Keep headers self-contained and prefer `#pragma once`; order function parameters with inputs before outputs.
- Apply clang-format or clang-tidy only to your edits and review diffs to avoid churn.

## Testing Guidelines
- After configuring, execute `cmake --build build --target check -j$(nproc)` or `cd build && make check` for the curated unit suites.
- `ctest -j$(nproc)` runs registered tests; add `-R <pattern>` or `-V` for focused, verbose runs. Failure logs land in `build/Testing/Temporary`.
- For integration smoke tests, build `vstart` and launch `./src/vstart.sh --debug --new -x --localhost --bluestore`; remember to stop with `./src/stop.sh`.
- Coordinate Teuthology suites under `qa/` and keep instructions with the scenario.

## Commit & Pull Request Guidelines
- Model commit subjects as `subsystem: concise summary` (see `git log`), keep them ≤72 characters, and include a descriptive body.
- Append `Signed-off-by: Full Name <email>` to each commit and link Ceph tracker issues via `Fixes:`/`Refs:` lines when applicable.
- Pull requests should list validation commands run, mention documentation or `PendingReleaseNotes` updates, and flag any follow-up work.
- Request reviews from subsystem maintainers in `src/OWNERS` or matching code owners, and prefer incremental commits until approval.
