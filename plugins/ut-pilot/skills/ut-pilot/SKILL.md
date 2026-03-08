---
name: ut-pilot
description: >
  Automated unit test writer and coverage improver for C/C++ projects using Google Test,
  Catch2, or similar frameworks. Analyzes source files, writes comprehensive test cases,
  integrates with CMake, and drives coverage toward >90%. Use this skill whenever the user
  wants to: write unit tests, increase test coverage, check coverage status, generate tests
  for untested files, or improve code coverage for any C/C++ project. Triggers on:
  "/ut-pilot:init", "/ut-pilot:status", "/ut-pilot:continue", "/ut-pilot:auto", "/ut-pilot:path",
  "write unit tests", "increase coverage", "add tests for", "coverage status",
  "what needs tests", "continue coverage", "generate tests", or any mention of improving
  C++ test coverage. Even if the user says "keep going" or "continue" in the context of
  UT work, use this skill.
---

# UT Pilot — Technical Reference

This skill provides shared technical knowledge for writing C++ unit tests. Command-specific
behavior (init, continue, auto, path, status) is defined in each command's `.md` file.
This file is referenced by agents spawned during parallel test writing.

---

## Write Test Workflow

### Step 1: Read and Understand the Source

**For files >200 lines**: Use LSP before reading the full file:
1. `document_symbols` on the `.hh`/`.cc` → get method list and signatures
2. From method names, identify which are public and have non-trivial implementation
3. Read only those sections (use offset+limit in the Read tool)
4. Focus test writing on those methods; skip private/unreachable utility code

This avoids reading thousands of lines to find 10 testable methods.

For all files, identify:

- **Class name and namespace**
- **Constructors**: default, parameterized, copy, move
- **Public API**: every public method signature
- **Enums and type aliases**
- **Dependencies**: what headers are included, what is forward-declared
- **Logging**: does it use `LOG_*`, spdlog, printf? (affects link dependencies)
- **Templates**: are there template specializations to test?
- **Static/free functions** in `.cc` files (need tests too)

If LSP is available:
- Use `document_symbols` to list all class methods without parsing headers manually
- Use `go to definition` on forward-declared types to confirm their actual type before writing tests
- Use `find references` to understand how a class is used across the project

### Step 2: Plan the Test File

Determine:

- **Test file path**: mirror source path under `test_root`
- **Naming**: apply `naming_convention` from `UT_RULES.md` (default: `{FileName}_ut.cc`)
- **Includes needed**: the header under test + dependency headers
  - If class A forward-declares class B, `#include "B.h"` BEFORE `#include "A.h"` in the test
- **Link dependencies**: list of `.cc` files that must be compiled together, and any libraries
- **CMake entry**: what to add to `CMakeLists.txt`

### Step 3: Write Comprehensive Tests

Target >90% line coverage. Structure:

```cpp
#include <gtest/gtest.h>

// Dependency headers BEFORE the file under test (resolves forward declarations)
#include "Dependency.h"
#include "FileUnderTest.h"

using namespace the_namespace;

// --- Construction ---
TEST(ClassNameTest, DefaultConstructor) {
  ClassName obj;
  EXPECT_EQ(obj.get_field(), default_value);
}

TEST(ClassNameTest, ParameterizedConstructor) {
  ClassName obj(arg1, arg2);
  EXPECT_EQ(obj.get_field(), arg1);
}

// --- Getters / Setters ---
TEST(ClassNameTest, SetAndGetField) {
  ClassName obj;
  obj.set_field(42);
  EXPECT_EQ(obj.get_field(), 42);
}

// --- Business Logic (primary coverage target) ---
TEST(ClassNameTest, MethodWithBranch_TruePath) {
  // Setup for the if-branch
  // Action
  // Verify
}

TEST(ClassNameTest, MethodWithBranch_FalsePath) {
  // Setup for the else-branch
  // Action
  // Verify
}

// --- Edge Cases ---
TEST(ClassNameTest, EmptyInput) { ... }
TEST(ClassNameTest, BoundaryValues) { ... }

// --- Enum Coverage ---
TEST(ClassNameTest, AllEnumValues) {
  // Test each enum value in switch statements to hit every case
}
```

**Coverage strategy:**

1. Business logic methods in `.cc` are where uncovered lines live — prioritize those over getters/setters
2. Hit every branch: write a separate test for each path through `if/else` and each `switch` case
3. Error paths: if there's a `LOG_ERROR` or early return, write a test that triggers it
4. Templates: instantiate with at least 2 concrete types
5. Do not rely on implementation details — test through the public API

### Step 4: Integrate with CMake

Read the existing `CMakeLists.txt` in the test directory and **match its style exactly**. If the project uses custom macros, use them.

**Standard patterns:**

Header-only (no `.cc` needed):
```cmake
add_executable(FileName_ut FileName_ut.cc)
target_include_directories(FileName_ut PRIVATE ${INCLUDE_DIRS})
target_link_libraries(FileName_ut GTest::gtest_main pthread)
add_test(NAME FileName_ut COMMAND FileName_ut)
```

With implementation file:
```cmake
add_executable(FileName_ut FileName_ut.cc ${SRC_ROOT}/FileName.cc ${SRC_ROOT}/Dep.cc)
target_include_directories(FileName_ut PRIVATE ${INCLUDE_DIRS})
target_link_libraries(FileName_ut GTest::gtest_main pthread)
add_test(NAME FileName_ut COMMAND FileName_ut)
```

Using project macros (`ut_add_module_sources` / `ut_add_test` if defined):
```cmake
ut_add_test(FileName_ut FileName_ut.cc)
ut_add_module_sources(FileName_ut ${SRC_ROOT}/FileName.cc)
target_link_libraries(FileName_ut GTest::gtest_main pthread)
```

If source is compiled into an OBJECT library or static lib target, link against that instead of re-listing `.cc` files.

### Step 5: Build and Verify

```bash
cd <test_build_dir> && cmake .. && make -j$(nproc) && ctest --output-on-failure
```

ALL tests must pass — not just new ones. Fix errors before moving on.

### Step 6: Check Coverage per File

```bash
cd <test_build_dir> && bash coverage.sh
```

For each file you wrote tests for: verify >90% line coverage. If below:
1. Check the HTML report for which lines are uncovered
2. Write targeted tests for those branches
3. Rebuild and recheck (max 2 iterations)

---

## Error Fix Reference

| Error | Root Cause | Fix |
|-------|-----------|-----|
| `error: incomplete type` | Forward-declared type used before its header is included | Add `#include "TypeName.h"` BEFORE the header under test |
| `undefined reference to 'ClassName::method'` | `.cc` not compiled | Add `ClassName.cc` to `target_sources` or the executable sources list |
| `undefined reference to 'google::...log...'` | Logging library not linked | Add `glog` or `spdlog` to `target_link_libraries` |
| `undefined reference to 'pthread_...'` | pthread not linked | Add `pthread` to `target_link_libraries` |
| `error: 'X' was not declared` | Missing include | Add the required `#include` |
| Segfault / ASAN null-deref | Dereferencing uninitialized pointer | Construct proper objects; don't pass nullptr unless testing null handling |
| `cannot instantiate abstract class` | Class has pure virtual methods | Create a minimal concrete subclass in the test file for testing |
| Test links but coverage is 0% | Build and test binary are different; coverage not enabled | Ensure test was built with `ENABLE_COVERAGE=ON` flag |

---

## Coverage Operations

### TODO.md format (produced by gen_todo.sh)

TODO.md is organized as directory sections. Entries:
- `- [x] Foo.cc (94% - covered)` — at ≥90%, skip
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` — below 90%, has uncovered line count
- `- [ ] Baz.cc (0%, ? uncov - no tests)` — no instrumented lines captured yet

The uncovered line count from TODO.md is the primary input for **complex-first** scoring.

### Regenerate coverage + TODO.md
```bash
cd <test_root> && bash coverage.sh       # builds, tests, generates coverage_filtered.info + TODO.md
cd <test_root> && bash gen_todo.sh       # re-parse existing coverage_filtered.info only (faster)
```

### Check a specific file's coverage
```bash
lcov --list <test_root>/build_cov/coverage_filtered.info | grep "FileName"
```

### View uncovered lines
Open `<test_root>/coverage_report/<module_path>/FileName.cc.gcov.html` in a browser, or:
```bash
gcov -b -c FileName.cc  # shows branch coverage in terminal
```

---

## Patterns for Specific C++ Constructs

### Data classes (structs/classes with getters/setters)
Write one test per field: set + get. Fast to write, guarantees at least some coverage.
Then focus on any validation logic in setters.

### Config/parameter classes
Test defaults in one test. Test each setter. Test any `validate()` or `apply()` method with
both valid and invalid inputs.

### Abstract base classes / interfaces
Create a minimal concrete implementation in the test file:
```cpp
class ConcreteImpl : public AbstractBase {
 public:
  void pure_virtual_method() override { /* minimal */ }
};
TEST(AbstractBaseTest, Construction) {
  ConcreteImpl impl;
  // test methods defined on AbstractBase
}
```

### Template classes
Instantiate with at least two types:
```cpp
TYPED_TEST_SUITE(ContainerTest, ::testing::Types<int, double>);
TYPED_TEST(ContainerTest, Insert) { ... }
```
Or just write explicit tests with two types if `TYPED_TEST` is overkill.

### Classes with file I/O
Use `std::tmpnam` or create files in `/tmp/`:
```cpp
TEST(FileReaderTest, ReadsCorrectly) {
  auto tmp = std::string(std::tmpnam(nullptr));
  { std::ofstream f(tmp); f << "test content\n"; }
  FileReader reader(tmp);
  EXPECT_EQ(reader.read_line(), "test content");
  std::remove(tmp.c_str());
}
```

### Classes with deep system dependencies (DB / full context)
If the constructor requires a database object that cannot be constructed in isolation:
- Write tests for any methods that ARE callable without the full system context
- Add an entry to `UT_RULES.md ## Max Coverage Files` documenting the max achievable coverage and reason:
  ```
  - FileName.cc (MaxCov: X%) — constructor dereferences PlacerDB immediately
  ```
- Do not leave the file with zero attempt — test what you can, document what you cannot

---

## Difficulty Classification

Use this to classify files for strategy scoring:

| Difficulty | Criteria |
|-----------|---------|
| **Easy** | Header-only (no `.cc`), or `.cc` <50 lines, few internal includes |
| **Medium** | Has `.cc`, 50-150 lines, <5 internal includes, no system-level deps |
| **Hard** | Has `.cc`, >150 lines, many internal includes, some system deps but mockable |
| **System-Dependent** | Requires database/full-system context in constructor, external services, or integration-level setup — test what you can, document MaxCov |

Complexity score for **complex-first**:
- Has `.cc`: +2
- `.cc` >100 lines: +1
- Uncovered lines above median for the module: +1
- Internal includes >6: +1 (capped at +2 total for this)
- System-level dependency: mark Skip
