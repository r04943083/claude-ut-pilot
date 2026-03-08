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
Agents MUST attempt Tier 3 (stub/fixture) before adding any entry here.
Only add entries when compilation itself fails (missing library not on this machine).
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
BUILD_DIR="$SCRIPT_DIR/build_cov"
REPORT_DIR="$SCRIPT_DIR/coverage_report"
# Set these for your project:
SOURCE_ROOT="<source_root>"
MODULE="<ModuleName>"

mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"
cmake .. -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
ctest --output-on-failure

mkdir -p "$REPORT_DIR"
# Step 1: baseline (zero for all instrumented lines)
lcov --capture --initial --directory . --output-file coverage_baseline.info
# Step 2: actual execution counts
lcov --capture --directory . --output-file coverage_test.info

# Step 3: zero-fill source files not compiled into any test binary
coverage_zero="coverage_zero.info"
> "$coverage_zero"
covered_files=$(grep '^SF:' coverage_test.info 2>/dev/null | sed 's/^SF://' | sort -u)
while IFS= read -r -d '' src_file; do
  if echo "$covered_files" | grep -qxF "$src_file"; then continue; fi
  echo "TN:" >> "$coverage_zero"
  echo "SF:${src_file}" >> "$coverage_zero"
  line_num=0
  while IFS= read -r _line; do
    line_num=$((line_num + 1))
    echo "DA:${line_num},0" >> "$coverage_zero"
  done < "$src_file"
  echo "LF:${line_num}" >> "$coverage_zero"
  echo "LH:0" >> "$coverage_zero"
  echo "end_of_record" >> "$coverage_zero"
done < <(find "$SOURCE_ROOT" -type f \( -name '*.hh' -o -name '*.h' -o -name '*.cc' \) -print0 | sort -z)

# Step 4: merge + extract
lcov --add-tracefile coverage_baseline.info \
     --add-tracefile coverage_test.info \
     --add-tracefile coverage_zero.info \
     --output-file coverage_merged.info
lcov --extract coverage_merged.info "*/${MODULE}/source/*" --output-file coverage_filtered.info

# HTML report
SOURCE_ROOT_REAL=$(realpath "$SOURCE_ROOT")
genhtml coverage_filtered.info --output-directory "$REPORT_DIR" --prefix "$SOURCE_ROOT_REAL"

# Generate TODO.md (includes ALL source files, even uninstrumented ones)
bash "$SCRIPT_DIR/gen_todo.sh" "$MODULE" "$SOURCE_ROOT"

echo "Coverage report: $REPORT_DIR/index.html"
lcov --summary coverage_filtered.info
```

**`gen_todo.sh`** in test_root (generates TODO.md — includes ALL source files, even those with no tests):
```bash
#!/bin/bash
# Usage: bash gen_todo.sh [MODULE] [SOURCE_ROOT]
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
MODULE="${1:-MyModule}"
SOURCE_ROOT="${2:-<source_root>}"
COV_INFO="${SCRIPT_DIR}/build_cov/coverage_filtered.info"
SOURCE_ROOT_REAL=$(realpath "$SOURCE_ROOT" 2>/dev/null || echo "$SOURCE_ROOT")
OUTPUT="${SCRIPT_DIR}/TODO.md"

declare -A COV_HIT COV_TOTAL
if [ -f "$COV_INFO" ]; then
  current_file=""
  hit=0; total=0
  while IFS= read -r line; do
    case "$line" in
      SF:*)
        current_file="${line#SF:}"
        current_file="${current_file#$SOURCE_ROOT/}"
        current_file="${current_file#$SOURCE_ROOT_REAL/}"
        hit=0; total=0 ;;
      DA:*)
        exec_count="${line#DA:}"; exec_count="${exec_count#*,}"; exec_count="${exec_count%%,*}"
        total=$((total + 1))
        [ "$exec_count" -gt 0 ] 2>/dev/null && hit=$((hit + 1)) ;;
      end_of_record)
        [ -n "$current_file" ] && [ "$total" -gt 0 ] && \
          COV_HIT["$current_file"]=$hit && COV_TOTAL["$current_file"]=$total
        current_file="" ;;
    esac
  done < "$COV_INFO"
fi

mapfile -t ALL_FILES < <(cd "$SOURCE_ROOT" && find . -type f \( -name '*.hh' -o -name '*.h' -o -name '*.cc' \) | sed 's|^\./||' | sort)
declare -A DIRS
for f in "${ALL_FILES[@]}"; do DIRS["$(dirname "$f")"]=1; done
mapfile -t SORTED_DIRS < <(printf '%s\n' "${!DIRS[@]}" | sort)

{
  echo "# ${MODULE} Unit Test Coverage TODO"
  echo ""; echo "Auto-generated by gen_todo.sh on $(date '+%Y-%m-%d %H:%M:%S')"; echo ""
  for dir in "${SORTED_DIRS[@]}"; do
    rel_dir="$dir"; [ -z "$rel_dir" ] || [ "$rel_dir" = "." ] && rel_dir="(root)"
    depth=$(echo "$rel_dir" | tr -cd '/' | wc -c); depth=$((depth + 2))
    printf '%0.s#' $(seq 1 $depth); echo " ${rel_dir}"; echo ""
    for f in "${ALL_FILES[@]}"; do
      [ "$(dirname "$f")" != "$dir" ] && continue
      fname="$(basename "$f")"
      total="${COV_TOTAL[$f]:-0}"; hit="${COV_HIT[$f]:-0}"
      if [ "$total" -eq 0 ]; then
        echo "- [ ] ${fname} (0%, ? uncov - no tests)"
      else
        pct=$((hit * 100 / total)); uncov=$((total - hit))
        [ "$pct" -ge 90 ] && echo "- [x] ${fname} (${pct}% - covered)" || \
          echo "- [ ] ${fname} (${pct}%, ${uncov} uncov - needs tests)"
      fi
    done
    echo ""
  done
} > "$OUTPUT"
echo "Generated ${OUTPUT} (${#ALL_FILES[@]} files)"
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
