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
3b. **Project root**: Detect from `git rev-parse --show-toplevel`, or parent of source_root
4. **Test root**: Check for `tests/ut/`, `test/ut/`, `tests/`, `test/`, `ut/`
5. **Naming convention**: Look at existing test files — do they use `_test.cc`, `_ut.cc`, `_test.cpp`?
6. **Coverage tooling**: Look for `.lcovrc`, `coverage.sh`, `lcov`, `gcovr`
7. **Simplest source file**: Find a .h/.cc pair with minimal dependencies for the sample test
8. **Prebuilt libraries**: Look for `build/lib/`, `build/release/lib/`, `install/lib/` for `.a` files. Count how many exist.
9. **Include complexity**: Count distinct `target_include_directories` or `-I` entries across existing CMakeLists.txt. If >10 unique dirs, flag as "complex include tree".

Also check if `UT_RULES.md` already exists (re-init scenario).

## Step 2: Confirm Configuration

First, print ALL auto-detected settings so the user can see them:

```
Auto-detected configuration:

| Setting           | Detected Value          | Notes                          |
|-------------------|------------------------|--------------------------------|
| source_root       | /path/to/src           | (from $ARGUMENTS or auto-scan) |
| project           | /path/to/project       | (git root or parent of source_root) |
| test_root         | /path/to/tests/ut      | (default, or existing test dir)|
| module            | ModuleName             | from CMake project()           |
| build_system      | cmake                  | CMakeLists.txt found           |
| framework         | gtest                  | (default or detected)          |
| coverage_tool     | lcov                   | lcov found on system           |
| naming_convention | {FileName}_ut.cc       | (default, or from existing tests) |
| parallel_agents   | 5                      | concurrent agents per batch    |
| files_per_batch   | 10                     | files processed per batch      |
| strategy          | complex-first          | complex-first / medium-first / simple-first |
| build_simplification | basic               | aggregator if prebuilt libs >5 or include dirs >10 |
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
# <ModuleName> Unit Test Rules

## Configuration
- source_root: <resolved_absolute_path>
- project: <resolved_project_root>
- test_root: <resolved_absolute_path>
- module: <ModuleName>
- framework: gtest
- coverage_tool: lcov
- naming_convention: {FileName}_ut.cc
- build_system: cmake
- parallel_agents: 5
- files_per_batch: 10
- strategy: complex-first
- build_simplification: basic
- current_focus: ""

## Directory Convention

Mirror the `src/` tree under `tests/ut/`:
```
src/<module>/subdir/  ->  tests/ut/<module>/subdir/
```

## CMake Patterns

(Populated after first continue run — documents the specific cmake macros used in this project)

## Project Gotchas

(Populated by agents as issues are encountered during test writing.
Entry formats:
  - [BuildFail] FileName.cc — compilation/link error after 3 fix attempts
  - [NoCode] FileName.cc — no executable code to instrument
  - Free-form gotcha notes for patterns, fixtures, and workarounds)

## Max Coverage Files

Files documented at their maximum achievable coverage. Auto/continue loops exclude these.
Only add an entry when: (1) the file compiles and tests run, (2) maximum coverage has been
reached, (3) no further strategy can improve it. Do NOT add [BuildFail] for these files.

## Test Design Guidelines

(Populated by agents with project-specific test patterns and workarounds)
```

If `UT_RULES.md` already exists, preserve any existing `## CMake Patterns`, `## Project Gotchas`, and `## Max Coverage Files` sections; only update `## Configuration`.

### 3b. Create Infrastructure Files (only if they don't exist)

Do NOT overwrite existing files. Check existence first.

**When `build_simplification: basic`** (default), generate **`CMakeLists.txt`** in test_root:
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

**When `build_simplification: aggregator`**, generate **three files** instead of one:

**1. `CMakeLists.txt`** in test_root:
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

# Aggregator target and helper macros
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/UTHelpers.cmake)
include(${CMAKE_CURRENT_SOURCE_DIR}/module_config.cmake)

# Add test subdirectories below
# add_subdirectory(submodule)
```

**2. `cmake/UTHelpers.cmake`** in test_root:
```cmake
# Reusable UT helper macros
# Backward-compatible macros
macro(ut_add_module_sources)
  cmake_parse_arguments(UMS "" "NAME;PREBUILT_LIB" "SOURCES;INCLUDES;LIBS" ${ARGN})

  if(NOT UMS_NAME)
    message(FATAL_ERROR "ut_add_module_sources: NAME is required")
  endif()

  set(_use_object OFF)

  if(ENABLE_COVERAGE)
    set(_use_object ON)
  else()
    if(UMS_PREBUILT_LIB AND EXISTS "${UMS_PREBUILT_LIB}")
      add_library(${UMS_NAME} STATIC IMPORTED)
      set_target_properties(${UMS_NAME} PROPERTIES
        IMPORTED_LOCATION "${UMS_PREBUILT_LIB}"
      )
      if(UMS_INCLUDES)
        set_target_properties(${UMS_NAME} PROPERTIES
          INTERFACE_INCLUDE_DIRECTORIES "${UMS_INCLUDES}"
        )
      endif()
      if(UMS_LIBS)
        set_target_properties(${UMS_NAME} PROPERTIES
          INTERFACE_LINK_LIBRARIES "${UMS_LIBS}"
        )
      endif()
    else()
      if(UMS_PREBUILT_LIB)
        message(WARNING "ut_add_module_sources(${UMS_NAME}): pre-built lib '${UMS_PREBUILT_LIB}' not found, falling back to OBJECT library")
      endif()
      set(_use_object ON)
    endif()
  endif()

  if(_use_object)
    if(UMS_SOURCES)
      add_library(${UMS_NAME} OBJECT ${UMS_SOURCES})
      if(UMS_INCLUDES)
        target_include_directories(${UMS_NAME} PRIVATE ${UMS_INCLUDES})
      endif()
      if(UMS_LIBS)
        # OBJECT libraries can carry link dependencies via INTERFACE
        target_link_libraries(${UMS_NAME} INTERFACE ${UMS_LIBS})
      endif()
    else()
      # Header-only: create INTERFACE library
      add_library(${UMS_NAME} INTERFACE)
      if(UMS_INCLUDES)
        target_include_directories(${UMS_NAME} INTERFACE ${UMS_INCLUDES})
      endif()
    endif()
  endif()
endmacro()

macro(ut_add_test)
  cmake_parse_arguments(UAT "" "NAME" "SOURCES;DEPENDS;INCLUDES;LIBS" ${ARGN})

  if(NOT UAT_NAME)
    message(FATAL_ERROR "ut_add_test: NAME is required")
  endif()
  if(NOT UAT_SOURCES)
    message(FATAL_ERROR "ut_add_test(${UAT_NAME}): SOURCES is required")
  endif()

  # Collect object files from OBJECT library dependencies
  set(_obj_sources "")
  set(_link_deps "")
  foreach(_dep ${UAT_DEPENDS})
    if(TARGET ${_dep})
      get_target_property(_dep_type ${_dep} TYPE)
      if(_dep_type STREQUAL "OBJECT_LIBRARY")
        list(APPEND _obj_sources $<TARGET_OBJECTS:${_dep}>)
        # Pull in interface link libs from the object lib
        get_target_property(_iface_libs ${_dep} INTERFACE_LINK_LIBRARIES)
        if(_iface_libs)
          list(APPEND _link_deps ${_iface_libs})
        endif()
      elseif(_dep_type STREQUAL "INTERFACE_LIBRARY")
        list(APPEND _link_deps ${_dep})
      else()
        # STATIC, STATIC IMPORTED, etc. - link directly
        list(APPEND _link_deps ${_dep})
      endif()
    else()
      message(WARNING "ut_add_test(${UAT_NAME}): dependency '${_dep}' is not a target")
    endif()
  endforeach()

  add_executable(${UAT_NAME} ${UAT_SOURCES} ${_obj_sources})

  if(UAT_INCLUDES)
    target_include_directories(${UAT_NAME} PRIVATE ${UAT_INCLUDES})
  endif()

  # Always link gtest and pthread
  set(_all_libs GTest::gtest_main pthread)
  if(_link_deps)
    list(APPEND _all_libs ${_link_deps})
  endif()
  if(UAT_LIBS)
    list(APPEND _all_libs ${UAT_LIBS})
  endif()

  target_link_libraries(${UAT_NAME} ${_all_libs})
  add_test(NAME ${UAT_NAME} COMMAND ${UAT_NAME})
endmacro()

# Zero-config test macro using aggregator target
# Usage:
#   ut_add_simple_test(NAME Foo_ut SOURCES Foo_ut.cc)
#   ut_add_simple_test(NAME Foo_ut SOURCES Foo_ut.cc COVERAGE_SOURCES /path/to/Foo.cc)
#   ut_add_simple_test(NAME Foo_ut SOURCES Foo_ut.cc COVERAGE_SOURCES /path/to/Foo.cc
#                      COVERAGE_COMPILE_OPTIONS -fopenmp -include /path/to/header.hh)
macro(ut_add_simple_test)
  cmake_parse_arguments(UAST "" "NAME" "SOURCES;COVERAGE_SOURCES;COVERAGE_COMPILE_OPTIONS" ${ARGN})

  # Coverage OBJECT library: recompile specific sources with instrumentation
  set(_uast_cov_objects "")
  if(ENABLE_COVERAGE AND UAST_COVERAGE_SOURCES)
    set(_uast_cov_lib ${UAST_NAME}_cov)
    add_library(${_uast_cov_lib} OBJECT ${UAST_COVERAGE_SOURCES})
    target_link_libraries(${_uast_cov_lib} PRIVATE ${PROJECT_UT_DEPS_TARGET})
    target_compile_options(${_uast_cov_lib} PRIVATE -O0 -g --coverage -fprofile-arcs -ftest-coverage)
    if(UAST_COVERAGE_COMPILE_OPTIONS)
      target_compile_options(${_uast_cov_lib} PRIVATE ${UAST_COVERAGE_COMPILE_OPTIONS})
    endif()
    set(_uast_cov_objects $<TARGET_OBJECTS:${_uast_cov_lib}>)
  endif()

  add_executable(${UAST_NAME} ${UAST_SOURCES} ${_uast_cov_objects})
  target_link_libraries(${UAST_NAME} PRIVATE
    ${PROJECT_UT_DEPS_TARGET}
    GTest::gtest_main
    pthread
  )
  add_test(NAME ${UAST_NAME} COMMAND ${UAST_NAME})
endmacro()
```

**3. `module_config.cmake`** in test_root:
```cmake
# Project-specific aggregator target for unit test dependencies
# Edit this file to match your project's include directories and libraries.
# For large projects, consider splitting into:
#   module_config.cmake    — path variables and this INTERFACE target
#   {module}_ut_targets.cmake — per-module OBJECT/IMPORTED targets
# Then include both: include(module_config.cmake) + include({module}_ut_targets.cmake)

set(PROJECT_UT_DEPS_TARGET <ModuleName>_ut_deps)
add_library(${PROJECT_UT_DEPS_TARGET} INTERFACE)

# --- Include directories ---
# Add all directories that test files need to compile.
# target_include_directories(${PROJECT_UT_DEPS_TARGET} INTERFACE
#   ${SRC_ROOT}
#   ${SRC_ROOT}/include
#   ${SRC_ROOT}/third_party
# )

# --- Prebuilt static libraries ---
# Use --start-group/--end-group for circular dependencies between .a files.
# Use --allow-multiple-definition so coverage OBJECT libs can override prebuilt symbols.
# set(BUILD_LIB_DIR "${SRC_ROOT}/../build/lib")
# target_link_libraries(${PROJECT_UT_DEPS_TARGET} INTERFACE
#   -Wl,--start-group
#   ${BUILD_LIB_DIR}/libmodule_a.a
#   ${BUILD_LIB_DIR}/libmodule_b.a
#   -Wl,--end-group
#   -Wl,--allow-multiple-definition
#   -Wl,--unresolved-symbols=ignore-all
# )

# --- System libraries ---
# target_link_libraries(${PROJECT_UT_DEPS_TARGET} INTERFACE
#   glog pthread
# )
```

Create the `cmake/` directory under test_root before writing `UTHelpers.cmake`.

**`run_tests.sh`** in test_root:
```bash
#!/bin/bash
set -e
cd "$(dirname "$0")"

UT_MODULE="${1:-<ModuleName>}"
shift 2>/dev/null || true

mkdir -p build && cd build
cmake .. -DUT_MODULE="${UT_MODULE}" "$@"
make -j$(nproc)
ctest --output-on-failure
```

**`coverage.sh`** in test_root:
```bash
#!/bin/bash
set -e
if ! command -v lcov &> /dev/null; then
  echo "ERROR: lcov not found. Install with: apt-get install lcov"
  exit 1
fi
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
BUILD_DIR="$SCRIPT_DIR/build_cov"
REPORT_DIR="$SCRIPT_DIR/coverage_report"
UT_MODULE="${1:-<ModuleName>}"
# Module-to-source-root mapping (add entries for each module)
declare -A MODULE_SOURCE_ROOTS=(
  [<ModuleName>]="<source_root>"
)
SOURCE_ROOT="${MODULE_SOURCE_ROOTS[$UT_MODULE]}"
if [ -z "$SOURCE_ROOT" ]; then
  echo "ERROR: No source root mapping for module '$UT_MODULE'."
  exit 1
fi
MODULE="$UT_MODULE"

mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"
cmake .. -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
make -j$(nproc)
ctest --output-on-failure

mkdir -p "$REPORT_DIR"
# Run lcov from inside a dedicated gcov temp dir so .gcov intermediates
# stay isolated and never litter the workspace root.
mkdir -p gcov_tmp
cd gcov_tmp
# Step 1: baseline (zero for all instrumented lines)
lcov --capture --initial --directory .. --output-file ../coverage_baseline.info
# Step 2: actual execution counts
lcov --capture --directory .. --output-file ../coverage_test.info
# Clean up gcov intermediates immediately after capture
rm -f *.gcov
cd ..

# Step 3: Generate synthetic zero tracefile for source files with no coverage data
#
# WHY only .cc files?
#   Header files (.hh/.h) with inline code are compiled as part of the
#   translation units that #include them.  gcov/lcov automatically generates
#   DA entries for their executable lines in coverage_test.info — no synthetic
#   data needed.
#
#   Header files that contain ONLY declarations have ZERO executable lines.
#   Generating synthetic DA:linenum,0 for every line would fake-inflate
#   uncovered counts and misleadingly report 0% for files with nothing to cover.
#
#   .cc files that are not compiled under coverage genuinely need the 0%
#   signal so developers know they lack test coverage.
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
done < <(find "$SOURCE_ROOT" -type f -name '*.cc' -print0 | sort -z)

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

**`gen_todo.sh`** in test_root (generates TODO.md — includes ALL source files, with status
annotations from UT_RULES.md for classified files):
```bash
#!/bin/bash
# Usage: bash gen_todo.sh [MODULE] [SOURCE_ROOT]
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
MODULE="${1:-MyModule}"
# Module-to-source-root mapping (fallback if arg 2 not provided)
declare -A MODULE_SOURCE_ROOTS=(
  [<ModuleName>]="<source_root>"
)

if [ -n "$2" ]; then
  SOURCE_ROOT="$2"
else
  SOURCE_ROOT="${MODULE_SOURCE_ROOTS[$MODULE]}"
  if [ -z "$SOURCE_ROOT" ]; then
    echo "ERROR: No source root mapping for module '$MODULE'. Provide as arg 2."
    exit 1
  fi
fi
COV_INFO="${SCRIPT_DIR}/build_cov/coverage_filtered.info"
SOURCE_ROOT_REAL=$(realpath "$SOURCE_ROOT" 2>/dev/null || echo "$SOURCE_ROOT")
OUTPUT="${SCRIPT_DIR}/TODO.md"
UT_RULES_FILE="${SCRIPT_DIR}/UT_RULES.md"

# Parse UT_RULES.md for status labels ([BuildFail], [NoCode], [DeclOnly], [MaxCov])
declare -A FILE_STATUS_LABEL  # basename -> annotation string
if [ -f "$UT_RULES_FILE" ]; then
  # Scan entire file for [BuildFail], [NoCode] single-line gotcha entries
  while IFS= read -r line; do
    if [[ "$line" =~ ^\-[[:space:]]\[(BuildFail|NoCode|DeclOnly)\][[:space:]]([[:alnum:]_.]+)[[:space:]]— ]]; then
      tag="${BASH_REMATCH[1]}"
      fname="${BASH_REMATCH[2]}"
      reason=$(echo "$line" | sed 's/.*— //')
      FILE_STATUS_LABEL["$fname"]="**[${tag}]** ${reason}"
    fi
  done < "$UT_RULES_FILE"
  # Scan for [MaxCov] entries (format: **[MaxCov] FileName (X%, Y uncov)**: reason)
  while IFS= read -r line; do
    if [[ "$line" =~ \[MaxCov\][[:space:]]+([[:alnum:]_.]+)[[:space:]]*\(([0-9]+)% ]]; then
      fname="${BASH_REMATCH[1]}"
      pct="${BASH_REMATCH[2]}"
      reason=$(echo "$line" | sed 's/.*\*\*: //' | sed 's/Maximum achievable.*//' | sed 's/[[:space:]]*$//')
      FILE_STATUS_LABEL["$fname"]="**[MaxCov ${pct}%]** ${reason}"
    fi
  done < "$UT_RULES_FILE"
fi

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
      status_label="${FILE_STATUS_LABEL[$fname]:-}"
      if [ "$total" -eq 0 ]; then
        # Headers with no executable code (declarations only) — skip silently.
        if [[ "$fname" == *.hh || "$fname" == *.h ]]; then
          continue
        elif [ -n "$status_label" ]; then
          echo "- [ ] ${fname} (0%, ? uncov — ${status_label})"
        else
          echo "- [ ] ${fname} (0%, ? uncov - no tests)"
        fi
      else
        pct=$((hit * 100 / total)); uncov=$((total - hit))
        if [ "$pct" -ge 90 ]; then
          echo "- [x] ${fname} (${pct}% - covered)"
        else
          if [ -n "$status_label" ]; then
            echo "- [ ] ${fname} (${pct}%, ${uncov} uncov — ${status_label})"
          else
            echo "- [ ] ${fname} (${pct}%, ${uncov} uncov - needs tests)"
          fi
        fi
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
3. Add a matching entry in CMakeLists.txt:
   - If `build_simplification: aggregator`, use `ut_add_simple_test(NAME FileName_ut SOURCES FileName_ut.cc)`
   - Otherwise, use the basic `ut_add_test` or manual pattern

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

(If build_simplification: aggregator)
- cmake/UTHelpers.cmake — helper macros including ut_add_simple_test
- module_config.cmake — aggregator target definition (edit to add your include dirs and libs)

Next step: /ut-pilot:continue
```
