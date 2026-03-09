# UT Pilot

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Language: C/C++](https://img.shields.io/badge/Language-C%2FC%2B%2B-blue.svg)
![Coverage Target: >90%](https://img.shields.io/badge/Coverage%20Target-%3E90%25-brightgreen.svg)

Automated C/C++ unit test writer and coverage driver for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

UT Pilot analyzes your source code, writes comprehensive Google Test (or Catch2) tests, integrates them into CMake, and iteratively drives line coverage toward >90%. It handles the hard parts: private member access, singleton stubbing, initialization chain tracing, OpenMP-safe test design, and dual fast/coverage build modes.

---

## Features

- Works with **Google Test**, **Catch2**, and other C++ test frameworks
- Supports **CMake** with automatic build system integration
- Generates **lcov/gcov** HTML coverage reports
- **File classification**: [NoCode], [DeclOnly], [BuildFail], [MaxCov] labels track what is and isn't coverable
- **Dual build modes**: fast iteration build (seconds) and full coverage instrumentation build (minutes)
- **Smart prioritization**: complex-first scoring picks the highest-impact uncovered files
- **Incremental**: run `/ut-pilot:continue` repeatedly to steadily increase coverage
- **Aggregator CMake pattern**: `ut_add_simple_test` macro eliminates per-test boilerplate
- **Singleton Stub Pattern**: lightweight no-op overrides for cross-module dependencies without full system init
- **Initialization chain tracing**: fixture segfaults are diagnosed and fixed rather than marked untestable
- **OpenMP-aware**: documents fork-safety constraints for threaded code that calls `exit()`

---

## Install

### From Claude Code Plugin Marketplace

```bash
# Step 1: Add the plugin source
/plugin marketplace add r04943083/claude-ut-pilot

# Step 2: Install the skill
/plugin install ut-pilot@claude-ut-pilot
```

### From GitHub (manual)

```bash
claude install-skill https://github.com/r04943083/claude-ut-pilot
```

### Local install

```bash
git clone https://github.com/r04943083/claude-ut-pilot.git
claude install-skill ./claude-ut-pilot
```

---

## Quick Start

After installing, open Claude Code in any C/C++ project:

```
> /ut-pilot:init
```

UT Pilot scans your project, sets up the test infrastructure, and writes your first test.

### Step 1: Initialize (first time only)

```
> /ut-pilot:init
```

Bootstraps the test infrastructure: creates the test directory, `CMakeLists.txt`, `run_tests.sh`, `coverage.sh`, `gen_todo.sh`, `UT_RULES.md`, `TODO.md`, and a sample test. Only needed once per project.

### Step 2: Check what needs testing

```
> /ut-pilot:status
```

Shows a coverage summary: how many files are tested, which directories need work, and the next targets sorted by impact.

### Step 3: Write tests

```bash
# Target a specific directory or file
> /ut-pilot:path src/core/parser/

# Or let UT Pilot auto-pick the highest-impact uncovered files
> /ut-pilot:continue
```

UT Pilot will:
1. Find uncovered source files
2. Read and understand each file's API
3. Write comprehensive unit tests
4. Add CMake build entries
5. Build and verify all tests pass
6. Report per-file coverage results

### Step 4: Repeat

```
> /ut-pilot:continue
> /ut-pilot:continue
> /ut-pilot:continue
...
```

Or run fully automatically until all files reach >90%:

```
> /ut-pilot:auto
```

---

## Architecture

### Plugin Structure

```
plugins/ut-pilot/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata (name, description, author)
├── commands/                # One .md file per user-facing command
│   ├── init.md              # /ut-pilot:init — bootstrap infrastructure
│   ├── status.md            # /ut-pilot:status — show coverage summary
│   ├── path.md              # /ut-pilot:path — focus on a directory or file
│   ├── continue.md          # /ut-pilot:continue — write next batch of tests
│   └── auto.md              # /ut-pilot:auto — loop continue to completion
└── skills/ut-pilot/
    └── SKILL.md             # Shared technical reference for all spawned agents
```

**Commands vs. Skills separation**: Each command `.md` file defines the high-level behavior for one user command (what to do, in what order). `SKILL.md` is the shared technical reference — it documents file classification rules, CMake patterns, C++ test techniques, and error fix recipes. Agents spawned during parallel test writing load `SKILL.md` as context so they apply consistent patterns without re-discovering them.

### Key Artifacts Generated Under `test_root`

| Artifact | Description |
|----------|-------------|
| `UT_RULES.md` | Project-specific rules: naming conventions, [NoCode]/[BuildFail]/[MaxCov] labels, gotchas |
| `TODO.md` | Per-file coverage status with uncovered line counts; drives the continue/auto loop |
| `CMakeLists.txt` | Test build system; uses `ut_add_simple_test` aggregator macro |
| `run_tests.sh` | Fast build + run (links prebuilt `.a`; builds in seconds) |
| `coverage.sh` | Coverage build + run + report generation (recompiles from source with gcov; takes minutes) |
| `gen_todo.sh` | Re-parses existing coverage data to refresh `TODO.md` without rebuilding |
| `cmake/UTHelpers.cmake` | Defines `ut_add_simple_test`, `ut_add_test`, `ut_add_module_sources` macros |
| `cmake/module_config.cmake` | INTERFACE aggregator target bundling all shared include dirs and prebuilt `.a` links |
| `build/` | Fast build output |
| `build_cov/` | Coverage build output + `.gcda`/`.gcno` files |
| `coverage_report/` | HTML report from genhtml |

### Dual Build Modes

| Mode | Script | Purpose | Build time |
|------|--------|---------|-----------|
| **Fast** | `run_tests.sh` | Build and run tests via prebuilt `.a` libraries — no instrumentation. Use for rapid iteration while writing tests. | Seconds |
| **Coverage** | `coverage.sh` | Passes `-DENABLE_COVERAGE=ON`. Recompiles source files listed in `COVERAGE_SOURCES` with gcov flags. Generates HTML report. | Minutes |

`gen_todo.sh` is a separate lightweight script that re-parses existing `.info` coverage data and refreshes `TODO.md` without any rebuild. Run it after updating labels in `UT_RULES.md` to reflect the changes immediately.

### File Classification System

Every source file is classified before test writing to determine the right action:

| Label | Meaning | Action |
|-------|---------|--------|
| `[NoCode]` | No executable code (commented-out bodies, empty class, pure enums, macros only) | Write one compile-verification test (`SUCCEED()`). Document in `UT_RULES.md`. |
| `[DeclOnly]` | Header with only declarations — no inline method bodies, no data-member initializers | Write instantiation/construction tests. gcov will show 0% (nothing to instrument), but tests still verify compile-time correctness. |
| `[BuildFail]` | Compilation or link fails after ≥3 fix attempts including the prebuilt approach | Document error summary in `UT_RULES.md`. Skip in continue/auto loop. |
| `[MaxCov]` | Compiles and runs, but coverage ceiling is hit due to architectural constraints | Document reason and achieved % in `UT_RULES.md ## Max Coverage Files`. Skip in continue/auto loop. |

`[NoCode]` and `[BuildFail]` are mutually exclusive with `[MaxCov]`. A file that compiles but has an exit() path cannot simultaneously be `[BuildFail]`.

### Coverage Prioritization Strategy

The continue/auto loop picks files using a **complex-first** scoring formula:

| Factor | Score |
|--------|-------|
| Has a `.cc` file | +2 |
| `.cc` is >100 lines | +1 |
| Uncovered lines above module median | +1 |
| Internal `#include`s >6 | +1 (capped at +2 total) |

The uncovered line count from `TODO.md` is the primary input. Files with more uncovered lines and higher complexity score are tackled first to maximize coverage gain per session.

The `/ut-pilot:path` command overrides prioritization and focuses all effort on a specific directory or file. This setting persists to `UT_RULES.md` as `current_focus`.

---

## Commands Reference

### `/ut-pilot:init`

Bootstrap UT infrastructure for a new C/C++ project. Run once per project.

- Discovers project layout: CMake structure, source root, build system, test frameworks
- Creates `test_root/` with `CMakeLists.txt`, shell scripts, and CMake helpers
- Generates `UT_RULES.md` with detected project conventions and naming rules
- Writes an initial `TODO.md` by scanning source files
- Writes a sample test to verify the build works end-to-end

### `/ut-pilot:status`

Show current test coverage status and next targets.

- Reads `TODO.md` to summarize per-directory and overall coverage
- Lists how many files are at ≥90%, how many need tests, and what the next targets are
- Reports [NoCode], [DeclOnly], [BuildFail], [MaxCov] file counts separately

### `/ut-pilot:path <path>`

Set the focused directory (or file) for the next `continue` run.

- Saves `current_focus = <path>` to `UT_RULES.md`
- Subsequent `/ut-pilot:continue` runs target only this path
- Useful for drilling into a specific module before resuming global coverage work

### `/ut-pilot:continue`

Auto-pick the next batch of uncovered files and write tests.

- Reads `TODO.md` sorted by complex-first score
- Skips files labeled [NoCode], [BuildFail], [MaxCov], or already at ≥90%
- For each picked file: reads source, classifies, writes test, updates CMake, builds, verifies coverage
- Updates `TODO.md` with new coverage numbers after each file

### `/ut-pilot:auto`

Loop `/ut-pilot:continue` automatically until all files reach >90% coverage or no more progress can be made. Equivalent to running `continue` in a loop. Reports a final summary when done.

---

## CMake Integration

### Basic Pattern (`ut_add_test`)

For tests with explicit source and dependency lists:

```cmake
ut_add_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  DEPENDS mymodule_sources
  INCLUDES ${INCLUDE_DIRS}
)
```

### Aggregator Pattern (`ut_add_simple_test`)

Zero-config test registration — the preferred approach when the aggregator target is set up:

```cmake
# Header-only or pure logic — no additional sources needed
ut_add_simple_test(NAME FileName_ut SOURCES FileName_ut.cc)
```

The macro automatically links the shared INTERFACE aggregator target (`myproject_ut_deps`) which bundles all include directories and prebuilt `.a` files for the project.

### `COVERAGE_SOURCES` — Required for Real Coverage

Without `COVERAGE_SOURCES`, the test exercises code through the prebuilt `.a` — which is not instrumented and yields **0% coverage** for that file.

```cmake
ut_add_simple_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyClass.cc   # required for real coverage
)
```

When `ENABLE_COVERAGE=ON` (i.e., during `coverage.sh`), a `MyClass_ut_cov` OBJECT library is created from `MyClass.cc` with gcov flags. It is linked before the prebuilt `.a` so the instrumented symbols win via `--allow-multiple-definition`.

With extra compile options (e.g., for OpenMP or force-included headers):

```cmake
ut_add_simple_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyClass.cc
  COVERAGE_COMPILE_OPTIONS -fopenmp -include ${SRC_ROOT}/util/usage.hh
)
```

### Stub Integration

Singleton stubs are added to a test target as additional private sources:

```cmake
ut_add_simple_test(
  NAME MyFile_ut
  SOURCES MyFile_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyFile.cc
)
target_sources(MyFile_ut PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/stubs/Singleton_stub.cc
)
```

### Linker Flags Reference

| Flag | Purpose |
|------|---------|
| `--start-group` / `--end-group` | Resolve circular dependencies between `.a` files |
| `--allow-multiple-definition` | Let coverage OBJECT symbols override prebuilt `.a` symbols |
| `--unresolved-symbols=ignore-all` | Partial linking when transitive dependencies are unavailable |

---

## C++ Test Techniques

### Private Member Access (`#define private public`)

Access all private/protected fields and methods directly in tests:

```cpp
#include <gtest/gtest.h>
#include <string>       // ALL system/STL headers BEFORE the #define block

#define private public
#define protected public
#include "MyClass.h"    // project headers AFTER the #define
#undef private
#undef protected
```

**Rules:**
- All `<system>` and third-party headers (`<gtest/gtest.h>`, `<string>`, `<vector>`) **must** come before `#define private public` — otherwise STL internals break
- `#undef` after project includes restores normal semantics for the rest of the file
- More portable than `-fno-access-control`; works with both GCC and Clang

This pattern enables direct member field access (`obj.field_`) and calling private methods without any public-API workarounds.

### Singleton Stub Pattern

When the file under test calls a singleton that requires full system initialization, create a lightweight stub override in `<test_root>/<module>/stubs/Singleton_stub.cc`:

```cpp
// Provides lightweight overrides for SomeSingleton methods called by the code under test.
// The --allow-multiple-definition linker flag lets this override the prebuilt .a version.
#include "SomeSingleton.hh"

namespace project {

SomeSingleton* SomeSingleton::_s_instance = nullptr;

SomeSingleton& SomeSingleton::getInst() {
  if (!_s_instance) _s_instance = new SomeSingleton();
  return *_s_instance;
}

void SomeSingleton::destroyInst() {
  delete _s_instance;
  _s_instance = nullptr;
}

// Override only the methods called by the file under test.
// Return values chosen so tests can reach all branches.
bool SomeSingleton::doOperation(Item* /*item*/) { return true; }
double SomeSingleton::queryValue(int /*id*/) { return 0.0; }

}  // namespace project
```

To cover both branches of `if (singleton.doOp())`, use a global flag in the stub:

```cpp
static bool g_stub_result = true;
bool SomeSingleton::doOperation(Item*) { return g_stub_result; }
// In test: g_stub_result = false; obj.method();  // → covers false branch
```

Always call `SomeSingleton::destroyInst()` in fixture `TearDown()` to prevent state leaks between tests.

**Key insight**: Cross-module singletons are ordinary C++ classes with source under the project root. They are not unresolvable system dependencies. Always try stubs or source compilation before reaching for prebuilt libraries (which yield 0% coverage).

### Fixture Initialization Chains

When a test segfaults with a null-pointer dereference, this almost always means a missing initialization step in `SetUp()` — not that the file is system-dependent or `[BuildFail]`.

**Debugging process:**
1. Identify the null pointer from the crash address or ASAN output
2. Find what field was null (e.g., `_manager` in `obj->_manager->method()`)
3. Search the source for the method that populates `_manager` (e.g., `initManager()`)
4. Add the missing call to `SetUp()`, and its inverse to `TearDown()`

```cpp
void SetUp() override {
  obj.initContextA();    // populates _context
  obj.initManager();     // populates _manager — was missing, caused segfault
}
void TearDown() override {
  obj.destroyManager();
  obj.destroyContextA(); // reverse order
}
```

Read the source file's own constructor or `init*()` methods to discover the required initialization sequence. The project's integration tests or main entry point often show the correct order.

### OpenMP / Threading Constraints

**Do NOT use `EXPECT_EXIT` for code that uses OpenMP or other fork-unsafe threading.**

When `EXPECT_EXIT` forks a child process, the child's OpenMP thread pool is absent. Any parallel section in the child deadlocks, hanging the parent wait forever.

If a code path calls `exit()` on failure AND uses OpenMP (e.g., numerical divergence detection in a solver), that path is fundamentally uncoverable via `EXPECT_EXIT`. Document it as `[MaxCov]`:

```
- **[MaxCov] NesterovPlace.cc (77%, 251 uncov)**: runNesterovPlace diverges+exit(1);
  EXPECT_EXIT incompatible with OpenMP fork-safety. Maximum achievable: 77%.
```

---

## Design Principles

### Source Compilation Principle

**If the project's main build succeeds, every `.cc` file under the project root can be recompiled in the UT build environment using the same include paths.**

This has two corollaries:

1. **Every `#include` is resolvable.** When an agent cannot find a header: do not mark the file as `[BuildFail]`. Search under the project path, add the directory to `target_include_directories`, and try again.

2. **Every `.cc` can be compiled from source.** Prefer compiling source files directly with `COVERAGE_SOURCES` over linking prebuilt libraries. Prebuilt = 0% coverage; source = real instrumentation data.

Do not give up on a file because it seems hard to compile. The project already proves the code compiles.

### Prebuilt Libraries as Last Resort

The prebuilt library strategy (linking `.a` without recompilation) yields **0% coverage** for the linked file. Use it only when:
- The dependency requires a binary-only external SDK
- The component's source is genuinely not under the project root
- Three fix strategies have failed including source recompilation

When forced to use prebuilt, record the file in `UT_RULES.md ## Max Coverage Files` and switch to `COVERAGE_SOURCES` as soon as source compilation becomes feasible.

### Cross-Module Dependencies

The escalation order for handling cross-module or singleton dependencies (stop when ≥90% coverage is reached):

1. **Reuse existing fixtures** — search `_ut.cc` files in the same module for MinimalWrapper / test-fixture classes that already set up the required context
2. **Write a singleton stub** — lightweight no-op overrides via `--allow-multiple-definition`
3. **Compile the dependency from source** — add its `.cc` files to `COVERAGE_SOURCES`
4. **Test pure-logic methods standalone** — static methods, pure-math utilities, and config getters need no system context
5. **Prebuilt library** (last resort) — only for truly unavailable binary-only external SDKs

---

## Requirements

- C/C++ project with CMake
- Google Test or Catch2 installed (or fetchable via FetchContent)
- lcov + gcov for coverage reports (optional but recommended)
- Claude Code CLI

---

## License

MIT
