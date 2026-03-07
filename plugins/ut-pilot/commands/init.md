---
description: "Bootstrap UT infrastructure for a new C/C++ project"
---

# /ut-pilot:init — Bootstrap UT Infrastructure

**Arguments**: `$ARGUMENTS` (optional path to source root or module directory)

## Step 1: Silent Auto-Discovery

Scan the project silently (no output yet). Detect:

1. **Build system**: Look for `CMakeLists.txt` (CMake), `Makefile` (Make), `BUILD`/`BUILD.bazel` (Bazel), `meson.build` (Meson)
2. **Existing test framework**: grep for `gtest`, `catch2`, `doctest`, `boost_test` in CMakeLists or source files
3. **Source root**: Check for `src/`, `source/`, `lib/` directories; if `$ARGUMENTS` is provided, use it directly
4. **Test root**: Check for `tests/ut/`, `test/ut/`, `tests/`, `test/`, `ut/`
5. **Naming convention**: Look at existing test files — do they use `_test.cc`, `_ut.cc`, `_test.cpp`?
6. **Coverage tooling**: Look for `.lcovrc`, `coverage.sh`, `lcov`, `gcovr`
7. **Simplest source file**: Find a .h/.cc pair with minimal dependencies for the sample test

Also check if `UT_RULES.md` already exists (re-init scenario).

## Step 2: Ask for Configuration Confirmation

First, print the detected configuration as a table:

```
I detected the following configuration:

| Setting           | Detected Value          | Notes                          |
|-------------------|------------------------|--------------------------------|
| source_root       | /path/to/src           | (from $ARGUMENTS or auto-scan) |
| test_root         | /path/to/tests/ut      |                                |
| module            | ModuleName             | top-level module name          |
| framework         | gtest                  |                                |
| coverage_tool     | lcov                   |                                |
| naming_convention | {FileName}_ut.cc       |                                |
| build_system      | cmake                  |                                |
| parallel_agents   | 3                      | agents to spawn per continue   |
| files_per_batch   | 5                      | files handled per continue     |
| strategy          | complex-first          | complex-first / medium-first / simple-first |
```

Then use the **`AskUserQuestion`** tool to ask:

> Please confirm this configuration, or tell me which values to change.

**Wait for the user's response before proceeding.** Do not create any files or run any commands until the user replies.

## Step 3: Apply Changes and Execute (after user confirms)

Parse the user's reply:
- "confirm" or "ok" or "yes" → use detected values as-is
- Any corrections mentioned → update those fields

Then execute in order:

### 3a. Generate UT_RULES.md

Write `UT_RULES.md` to the test_root directory (or project root if test_root doesn't exist yet):

```markdown
## Configuration
- source_root: <resolved_absolute_path>
- test_root: <resolved_absolute_path>
- module: <ModuleName>
- framework: gtest
- coverage_tool: lcov
- naming_convention: {FileName}_ut.cc
- build_system: cmake
- parallel_agents: 3
- files_per_batch: 5
- strategy: complex-first
- current_focus: ""

## Test Conventions
(Populated after first continue run based on project patterns discovered)

## Project Gotchas
(Populated by agents as issues are encountered during test writing)
```

If `UT_RULES.md` already exists, preserve any existing `## Test Conventions` and `## Project Gotchas` sections; only update `## Configuration`.

### 3b. Create Infrastructure Files (only if they don't exist)

Do NOT overwrite existing files. Check existence first.

**`CMakeLists.txt`** in test_root:
```cmake
cmake_minimum_required(VERSION 3.14)
project(<ModuleName>_UnitTests)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(ENABLE_COVERAGE "Enable coverage instrumentation" OFF)

if(ENABLE_COVERAGE)
  add_compile_options(-O0 -g --coverage -fprofile-arcs -ftest-coverage)
  add_link_options(--coverage)
endif()

find_package(GTest REQUIRED)
include(CTest)
enable_testing()

# Source root (adjust if needed)
set(SRC_ROOT <source_root>)
set(INCLUDE_DIRS <source_root>)

# Convenience macros — match the style used in existing test directories if any
macro(ut_add_module_sources TARGET)
  target_sources(${TARGET} PRIVATE ${ARGN})
endmacro()

macro(ut_add_test NAME)
  add_executable(${NAME} ${ARGN})
  target_include_directories(${NAME} PRIVATE ${INCLUDE_DIRS})
  target_link_libraries(${NAME} GTest::gtest_main pthread)
  add_test(NAME ${NAME} COMMAND ${NAME})
endmacro()

# Add test subdirectories below
# add_subdirectory(submodule)
```

**`run_tests.sh`** in test_root:
```bash
#!/bin/bash
set -e
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
BUILD_DIR="$SCRIPT_DIR/build"
mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"
cmake .. -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
ctest --output-on-failure
```

**`coverage.sh`** in test_root:
```bash
#!/bin/bash
set -e
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
BUILD_DIR="$SCRIPT_DIR/build_coverage"
REPORT_DIR="$SCRIPT_DIR/coverage_report"

mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"
cmake .. -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
ctest --output-on-failure

mkdir -p "$REPORT_DIR"
lcov --capture --directory . --output-file coverage.info --rc lcov_branch_coverage=1
lcov --remove coverage.info '/usr/*' '*/gtest/*' '*/gmock/*' --output-file coverage_filtered.info
genhtml coverage_filtered.info --output-directory "$REPORT_DIR" --branch-coverage

# Generate TODO.md
python3 "$SCRIPT_DIR/gen_todo.py" coverage_filtered.info "$SCRIPT_DIR/TODO.md" || \
  lcov --list coverage_filtered.info > "$SCRIPT_DIR/coverage_summary.txt"

echo "Coverage report: $REPORT_DIR/index.html"
```

**`gen_todo.py`** in test_root (generates TODO.md from lcov info):
```python
#!/usr/bin/env python3
"""Parse lcov .info file and generate TODO.md listing files needing tests."""
import sys, re, os

def parse_lcov(info_file):
    files = {}
    current = None
    with open(info_file) as f:
        for line in f:
            line = line.strip()
            if line.startswith('SF:'):
                current = line[3:]
                files[current] = {'lines_found': 0, 'lines_hit': 0}
            elif line.startswith('LF:') and current:
                files[current]['lines_found'] = int(line[3:])
            elif line.startswith('LH:') and current:
                files[current]['lines_hit'] = int(line[3:])
    return files

def main():
    if len(sys.argv) < 3:
        print("Usage: gen_todo.py coverage.info TODO.md")
        sys.exit(1)
    info_file, todo_file = sys.argv[1], sys.argv[2]
    files = parse_lcov(info_file)

    needs_tests = []
    covered = []
    for path, data in sorted(files.items()):
        found = data['lines_found']
        hit = data['lines_hit']
        if found == 0:
            continue
        pct = hit / found * 100
        if pct < 90:
            needs_tests.append((path, pct, found - hit))
        else:
            covered.append((path, pct))

    with open(todo_file, 'w') as f:
        f.write("# TODO: Files Needing Tests\n\n")
        f.write(f"Generated from coverage data. {len(covered)} files at >90%, {len(needs_tests)} need work.\n\n")
        f.write("## Needs Tests\n\n")
        f.write("| File | Coverage | Uncovered Lines |\n")
        f.write("|------|----------|------------------|\n")
        for path, pct, uncov in sorted(needs_tests, key=lambda x: -x[2]):
            f.write(f"| {path} | {pct:.1f}% | {uncov} |\n")
        f.write("\n## Covered (>90%)\n\n")
        for path, pct in covered:
            f.write(f"- {path} ({pct:.1f}%)\n")
    print(f"TODO.md written: {len(needs_tests)} files need tests")

if __name__ == '__main__':
    main()
```

### 3c. Create Sample Test

Find the simplest source file (smallest .cc file with a corresponding .h, fewest includes):
1. Read the .h and .cc
2. Write a minimal test file at `<test_root>/<mirrored_path>/<FileName>_ut.cc`
3. Add a matching entry in CMakeLists.txt

### 3d. Verify Infrastructure

```bash
cd <test_root> && bash run_tests.sh
```

If it fails, diagnose and fix before proceeding.

### 3e. Generate Initial TODO.md

```bash
cd <test_root> && bash coverage.sh
```

This produces `TODO.md` with the full list of files needing tests.

### 3f. Final Report

```
Infrastructure ready:
- UT_RULES.md written to <test_root>/UT_RULES.md
- CMakeLists.txt, run_tests.sh, coverage.sh created
- Sample test: <test_root>/path/to/SampleFile_ut.cc (PASSING)
- TODO.md: <N> files need tests

Next step: /ut-pilot:continue
```
