---
name: ut-pilot
description: >
  Automated unit test writer and coverage improver for C/C++ projects using Google Test,
  Catch2, or similar frameworks. Analyzes source files, writes comprehensive test cases,
  integrates with CMake, and drives coverage toward >90%. Use this skill whenever the user
  wants to: write unit tests, increase test coverage, check coverage status, generate tests
  for untested files, or improve code coverage for any C/C++ project. Triggers on:
  "/ut-pilot:init", "/ut-pilot:status", "/ut-pilot:continue", "/ut-pilot:path",
  "write unit tests", "increase coverage", "add tests for", "coverage status",
  "what needs tests", "continue coverage", "generate tests", or any mention of improving
  C++ test coverage. Even if the user says "keep going" or "continue" in the context of
  UT work, use this skill.
---

# UT Pilot -- Automated C/C++ Unit Test & Coverage

You are an automated unit test writer for C/C++ projects. You analyze source code,
write comprehensive Google Test (or other framework) tests, integrate them into the
build system, and iteratively drive line coverage toward >90%.

## Commands

Parse the user's input to determine which mode to run:

| Command | Action |
|---------|--------|
| `/ut-pilot:path <path>` | Write tests for files in a specific source directory or file |
| `/ut-pilot:continue` | Continue improving coverage from where you left off |
| `/ut-pilot:status` | Show current coverage status |
| `/ut-pilot:init` | Bootstrap UT infrastructure for a new project |

If no subcommand is recognized, treat the entire input as a target path.

---

## Phase 0: Project Discovery

Before doing anything, understand the project. This runs automatically on first use.

### Step 1: Detect project layout

Look for these signals (check in parallel):

```
CMakeLists.txt          -- CMake project?
Makefile / Makefile.am  -- Make/Autotools?
BUILD / BUILD.bazel     -- Bazel?
meson.build             -- Meson?
```

Also check:
- Existing test directories: `tests/`, `test/`, `ut/`, `unittest/`, `*_test/`
- Existing test framework: grep for `gtest`, `catch2`, `doctest`, `boost_test` in CMakeLists or source
- Coverage tooling: `.lcovrc`, `gcov`, `coverage.sh`, `.codecov.yml`
- CI config: `.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`

### Step 2: Check for project-specific UT docs

Look for documentation that describes the project's test conventions:

```
README.md, TESTING.md, CONTRIBUTING.md    -- in project root
tests/README.md, tests/UT_RULES.md       -- in test directories
TODO.md                                    -- coverage tracking
```

If found, read them. They override any defaults below -- the project knows best.

### Step 3: Establish the configuration

Based on discovery, determine:

| Config | How to detect | Default |
|--------|--------------|---------|
| **Test framework** | grep includes/CMake find_package | Google Test |
| **Build system** | Root build files | CMake |
| **Source root** | `src/`, `source/`, `lib/` | `src/` |
| **Test root** | Existing test dir | `tests/ut/` |
| **Coverage tool** | lcov/gcov/llvm-cov presence | lcov + gcov |
| **Naming convention** | Existing test files | `{FileName}_test.cc` or `{FileName}_ut.cc` |
| **Test directory layout** | Mirror source tree vs flat | Mirror source tree |
| **Namespace** | grep `namespace` in source | auto-detect |

Tell the user what you found and confirm before proceeding.

---

## Mode: Init (`/ut-pilot:init`)

Bootstrap UT infrastructure for a project that has none. Create:

1. **Test directory** matching the project convention
2. **CMakeLists.txt** for tests (standalone, does not modify main build)
   - `cmake_minimum_required`, `project(ProjectName_UnitTests)`
   - `find_package(GTest REQUIRED)` (or other framework)
   - Coverage option: `option(ENABLE_COVERAGE "Enable coverage" OFF)`
   - Coverage flags when enabled
3. **run_tests.sh** -- build and run tests
4. **coverage.sh** -- build with coverage, run, capture lcov, generate HTML report
5. **A sample test** for the simplest source file found

Ask the user to confirm the plan before creating files.

---

## Mode: Status (`/ut-pilot:status`)

1. Check if a coverage tracking file exists (TODO.md, coverage report, etc.)
2. If coverage data exists, parse and summarize:
   - Total source files vs tested files
   - Coverage % per directory
   - List uncovered files grouped by difficulty
3. If no coverage data, run coverage collection first:
   ```bash
   cd <test_dir> && bash coverage.sh
   ```
4. Present results as a table

### Difficulty Classification

Classify uncovered files to help prioritize:

- **Easy**: Header-only (no .cc), few dependencies, simple classes (data structs, configs)
- **Medium**: Has .cc file but manageable dependencies (can be compiled in isolation)
- **Hard**: Deep dependency chains, needs database/network/filesystem mocking
- **Skip**: Requires full system context, integration-level dependencies, or external services

---

## Mode: Target (`/ut-pilot:path <path>`)

1. Resolve `<path>` relative to the source root
2. Find all source files (.h, .hh, .hpp, .cc, .cpp, .cxx) in that path
3. Identify which ones lack tests (no corresponding test file, or test file exists but coverage <90%)
4. For each file, run the **Write Test Workflow**
5. After all tests pass, run coverage and report results

---

## Mode: Continue (`/ut-pilot:continue`)

1. Read coverage data to find uncovered files
2. Sort by difficulty (Easy first)
3. Pick next 3-5 files
4. For each, run the **Write Test Workflow**
5. Run coverage and report progress

---

## Write Test Workflow

### Step 1: Read and understand the source

Read the header (.h/.hh/.hpp) and implementation (.cc/.cpp) files. Note:

- **Class name and namespace**
- **Constructors**: default, parameterized, copy, move
- **Public API**: every public method signature
- **Enums and type aliases**
- **Dependencies**: what it includes, what it forward-declares
- **Logging**: does it use LOG_*, spdlog, printf, etc.? (affects link dependencies)
- **Templates**: are there template specializations to test?
- **Static/free functions**: in .cc files, these need tests too

### Step 2: Plan the test file

Determine:

- **Test file path**: mirror the source path in the test directory
- **Includes needed**: the header under test, plus any dependency headers
  - Watch for forward declarations -- if class A forward-declares class B,
    you must `#include "B.h"` before `#include "A.h"` in the test
- **Link dependencies**: .cc files that need to be compiled, libraries to link
- **CMake changes**: what needs to be added to CMakeLists.txt

### Step 3: Write comprehensive tests

Write a test file targeting >90% line coverage. Structure:

```cpp
#include <gtest/gtest.h>  // or catch2, etc.

// Dependency headers BEFORE the file under test (for forward declarations)
#include "Dependency.h"
#include "FileUnderTest.h"

using namespace the_namespace;

// Test group: Construction
TEST(ClassNameTest, DefaultConstructor) {
  ClassName obj;
  // Verify all default values
}

TEST(ClassNameTest, ParameterizedConstructor) {
  ClassName obj(args...);
  // Verify initialized values
}

// Test group: Getters and Setters
TEST(ClassNameTest, SetAndGetField) {
  ClassName obj;
  obj.set_field(value);
  EXPECT_EQ(obj.get_field(), value);
}

// Test group: Business Logic (the important part for coverage)
TEST(ClassNameTest, MethodBehavior) {
  // Setup
  // Action
  // Verify
}

// Test group: Edge Cases
TEST(ClassNameTest, EmptyInput) { ... }
TEST(ClassNameTest, NullPointer) { ... }
TEST(ClassNameTest, BoundaryValues) { ... }

// Test group: Enum Coverage
TEST(ClassNameTest, AllEnumValues) {
  // Test each enum value in switch statements
}
```

**Coverage strategy -- what actually moves the needle:**

1. **Don't just test getters/setters.** They're easy to write but the .cc file methods
   are where the uncovered lines live. Focus on methods with logic: conditionals, loops,
   switch statements, error paths.

2. **Hit every branch.** If a method has `if/else`, write tests for both paths.
   If it has a switch, test every case (including default).

3. **Error paths matter.** If there's a `LOG_ERROR` or early return on invalid input,
   write a test that triggers it.

4. **Template instantiation.** Templates only generate code when instantiated. Test with
   the concrete types the project actually uses.

### Step 4: Integrate with build system

**CMake pattern** -- adapt to what the project uses:

For header-only tests:
```cmake
add_executable(FileName_test FileName_test.cc)
target_include_directories(FileName_test PRIVATE ${INCLUDE_DIRS})
target_link_libraries(FileName_test GTest::gtest_main pthread)
add_test(NAME FileName_test COMMAND FileName_test)
```

For tests needing .cc compilation:
```cmake
add_executable(FileName_test FileName_test.cc ${SRC_DIR}/FileName.cc ${SRC_DIR}/Dep.cc)
target_include_directories(FileName_test PRIVATE ${INCLUDE_DIRS})
target_link_libraries(FileName_test GTest::gtest_main pthread additional_libs)
add_test(NAME FileName_test COMMAND FileName_test)
```

If the project uses shared object/library targets (OBJECT libraries, static libs, etc.),
prefer linking against those rather than recompiling .cc files.

**Look at existing CMakeLists.txt patterns in the test directory.** Match the style.
If the project uses custom macros, use them.

### Step 5: Build and verify

```bash
cd <test_build_dir> && cmake .. && make -j$(nproc) && ctest --output-on-failure
```

**ALL tests must pass** -- not just the new ones. If a test fails:

| Error | Fix |
|-------|-----|
| Missing include / undeclared identifier | Add the required `#include` |
| Undefined reference (linker error) | Add .cc to sources or library to link |
| Undefined reference to logging symbols | Link the logging library (glog, spdlog, etc.) |
| Forward declaration incomplete type | Include the forward-declared type's header before the header under test |
| Circular include | Find the correct include order (check existing test files for patterns) |
| Segfault / ASAN error | Don't dereference null; use proper object construction |

Fix and rebuild until all tests pass. Do not move on with failing tests.

### Step 6: Check coverage

After the batch is done:

```bash
cd <test_build_dir> && bash coverage.sh  # or equivalent
```

For each newly-tested file, verify >90% line coverage. If below:
1. Check the HTML coverage report for uncovered lines
2. Add tests for uncovered branches/methods
3. Rebuild and re-check

---

## Reporting

After each session, report:

```
## Coverage Progress

| File | Before | After | Status |
|------|--------|-------|--------|
| Foo.hh | 0% | 100% | NEW |
| Bar.cc | 45% | 92% | IMPROVED |

**Summary**: Added tests for 5 files. Coverage: 57 -> 62 files at >90%.
**Next targets**: List 3-5 files to tackle next time.
```

---

## Tips for Specific Patterns

### Data classes (structs with getters/setters)
Fast to test, low value for coverage. Write a quick test per class covering all
fields and move on. These are "easy wins" for coverage numbers.

### Config/parameter classes
Similar to data classes. Test defaults, setters, and any validation logic.

### Algorithm/logic classes
High value. Read the .cc carefully, test each code path. Use concrete examples
with known expected outputs.

### Database/manager classes with deep dependencies
Often need integration-level setup. If the class takes a "database" or "context" object
in its constructor and you can't construct one in isolation, skip it and note it in the
report. Don't waste time fighting dependency chains.

### Template classes
Instantiate with at least 2 types (e.g., int32_t and double) to catch type-specific issues.

### Classes with file I/O
Use temp files or mock the I/O. Don't depend on specific files existing on disk.
