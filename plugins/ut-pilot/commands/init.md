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

## Step 2: Confirm Configuration

First, print ALL auto-detected settings so the user can see them:

```
Auto-detected configuration:

| Setting           | Detected Value          | Notes                          |
|-------------------|------------------------|--------------------------------|
| source_root       | /path/to/src           | (from $ARGUMENTS or auto-scan) |
| test_root         | /path/to/tests/ut      | (default, or existing test dir)|
| module            | ModuleName             | from CMake project()           |
| build_system      | cmake                  | CMakeLists.txt found           |
| framework         | gtest                  | (default or detected)          |
| coverage_tool     | lcov                   | lcov found on system           |
| naming_convention | {FileName}_ut.cc       | (default, or from existing tests) |
| parallel_agents   | 5                      | concurrent agents per batch    |
| files_per_batch   | 10                     | files processed per batch      |
| strategy          | complex-first          | complex-first / medium-first / simple-first |
```

Then use **one** `AskUserQuestion` call to ask these 4 key settings:

1. **source_root** — `<detected_path> (Recommended)`, plus any other candidate directories found (e.g., `source/`, `lib/`). Header: "Source root"
2. **test_root** — `<detected_path> (Recommended)`, plus alternatives like `tests/`, `test/ut/`, `test/`. Header: "Test root"
3. **parallel_agents** — `5 (Recommended)`, `1`, `3`. Header: "Agents"
4. **files_per_batch** — `10 (Recommended)`, `5`, `15`. Header: "Batch size"

**Wait for user response.** Apply any changes to the detected values.

Then print the final confirmed configuration table and continue to Step 3.

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
- parallel_agents: 5
- files_per_batch: 10
- strategy: complex-first
- current_focus: ""

## CMake Patterns
(Populated after first continue run — reference for the specific project's cmake macros)

## Project Gotchas
(Populated by agents as issues are encountered during test writing)

## Max Coverage Files
Files documented at their maximum achievable coverage without full system init:
(Agents add entries here when a file cannot reach >90% due to system dependencies)
```

If `UT_RULES.md` already exists, preserve any existing `## CMake Patterns`, `## Project Gotchas`, and `## Max Coverage Files` sections; only update `## Configuration`.

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

# Use realpath so --prefix strips correctly even if SOURCE_ROOT contains symlinks
SOURCE_ROOT_REAL=$(realpath "$SOURCE_ROOT")
genhtml coverage_filtered.info --output-directory "$REPORT_DIR" --branch-coverage --prefix "$SOURCE_ROOT_REAL"

# Generate TODO.md
python3 "$SCRIPT_DIR/gen_todo.py" coverage_filtered.info "$SCRIPT_DIR/TODO.md" || \
  lcov --list coverage_filtered.info > "$SCRIPT_DIR/coverage_summary.txt"

echo "Coverage report: $REPORT_DIR/index.html"
```

**`gen_todo.py`** in test_root (generates TODO.md from lcov info):
```python
#!/usr/bin/env python3
"""Parse lcov .info file and generate TODO.md in directory-organized bullet list format."""
import sys, os
from datetime import datetime
from collections import defaultdict

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
        print("Usage: gen_todo.py coverage.info TODO.md [source_root]")
        sys.exit(1)
    info_file, todo_file = sys.argv[1], sys.argv[2]
    source_root = sys.argv[3].rstrip('/') if len(sys.argv) > 3 else ''

    files = parse_lcov(info_file)

    # Strip source_root prefix to get relative paths
    rel_files = {}
    for path, data in files.items():
        rel = path[len(source_root)+1:] if source_root and path.startswith(source_root + '/') else path
        rel_files[rel] = data

    # Group by directory
    by_dir = defaultdict(list)
    for rel_path, data in sorted(rel_files.items()):
        d = os.path.dirname(rel_path)
        by_dir[d].append((os.path.basename(rel_path), rel_path, data))

    now = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    with open(todo_file, 'w') as f:
        f.write(f"# Unit Test Coverage TODO\n\nAuto-generated on {now}\n\n")
        for d in sorted(by_dir.keys()):
            depth = d.count('/') + 2 if d else 1
            heading = '#' * depth
            label = d if d else '(root)'
            f.write(f"{heading} {label}\n\n")
            for fname, rel_path, data in sorted(by_dir[d]):
                found = data['lines_found']
                hit = data['lines_hit']
                if found == 0:
                    f.write(f"- [ ] {fname} (0%, ? uncov - no tests)\n")
                else:
                    pct = int(hit / found * 100)
                    uncov = found - hit
                    if pct >= 90:
                        f.write(f"- [x] {fname} ({pct}% - covered)\n")
                    else:
                        f.write(f"- [ ] {fname} ({pct}%, {uncov} uncov - needs tests)\n")
            f.write("\n")
    print(f"TODO.md written: {len(rel_files)} files")

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
