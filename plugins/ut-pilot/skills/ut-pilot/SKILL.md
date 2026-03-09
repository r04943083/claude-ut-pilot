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

## Source Root Policy

`source_root` is a **read-only reference path**. Agents use it to locate and read source
files, resolve `#include` paths, and enumerate coverage targets. **Never write any files
into `source_root`.**

All generated artifacts belong under `test_root`:
- `UT_RULES.md`, `TODO.md` — test_root root
- `CMakeLists.txt`, `run_tests.sh`, `coverage.sh`, `gen_todo.sh` — test_root root
- `cmake/UTHelpers.cmake`, `module_config.cmake` — test_root root
- `build/`, `build_cov/`, `coverage_report/` — test_root subdirectories
- Test source files (`*_ut.cc`) — test_root mirroring source tree structure

---

## File Classification

**Classify every file before writing tests.** This prevents incorrect [BuildFail] labels
and determines the right action for each file type.

### [NoCode] — No Executable Code

Applies when the file contains nothing that gcov can instrument:
- All method bodies are commented out
- Empty class body (`class Foo {};`)
- Pure enum or enum class definitions
- Only macros, typedefs, or `#include` directives

**Action**: Write one compile-verification test (`TEST(X, Compiles) { SUCCEED(); }`).
Record in `UT_RULES.md ## Project Gotchas`:
```
- [NoCode] FileName — reason (e.g., all method bodies commented out)
```

### [DeclOnly] — Declaration-Only Header

Applies to `.hh`/`.h` files containing only class declarations and method signatures,
with no inline method bodies (`{ ... }`) and no data-member initializers with values.

gcov does not instrument pure declarations, but these headers are still **testable** —
instantiation tests verify constructors, default values, and type correctness. Use the
`#define private public` pattern to access all members.

**Action**: Write instantiation/construction tests:
```cpp
TEST(ClassNameTest, DefaultConstruction) {
  ClassName obj;
  // verify default member values, sizeof, type traits
}
TEST(ClassNameTest, FieldAccess) {
  ClassName obj;
  obj.field_ = value;  // direct access via #define private public
  EXPECT_EQ(obj.field_, value);
}
```
Record in `## Project Gotchas` as `- [DeclOnly] FileName.hh — reason`.
This documents why the file has no coverage data. `[DeclOnly]` files are NOT excluded
from the continue/auto loop — agents should write instantiation tests for them.

### [BuildFail] — Compilation Fails

**Strict definition** — only add `[BuildFail]` when ALL of the following hold:
- You actually ran `bash run_tests.sh` (or equivalent cmake command)
- The build returned non-zero due to a **compiler or linker error** (not a runtime crash)
- You tried at least 3 different fix strategies, including the prebuilt library approach

**Not [BuildFail]**:
- The test binary runs but crashes at runtime → try fixture/stub or prebuilt library approach
- Coverage is low or zero → use [MaxCov] instead
- You only read the code and inferred it might not compile → actually run cmake first

Record in `UT_RULES.md ## Project Gotchas`:
```
- [BuildFail] FileName.cc — <concise error summary>
```

### [MaxCov] — Coverage Ceiling

When a file compiles and runs but coverage cannot be improved further due to architectural
constraints (e.g., private initialization gates, legacy dead-code paths, external service
dependencies that cannot be stubbed):

**Action**: Record in `UT_RULES.md ## Max Coverage Files`:
```
- **[MaxCov] FileName.cc (X%, Y uncov)**: reason. Maximum achievable: X%.
```

Do **not** also add a `[BuildFail]` entry — these two labels are mutually exclusive.
Files with [MaxCov] are excluded by the auto/continue loop just like [BuildFail] files.

---

## Write Test Workflow

### Step 1: Read and Understand the Source

**For files >200 lines**: Use LSP before reading the full file:
1. `document_symbols` on the `.hh`/`.cc` → get method list and signatures
2. From method names, identify methods with non-trivial implementation
3. Read only those sections (use offset+limit in the Read tool)
4. Focus test writing on those methods — private methods are accessible via `#define private public`

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

**Tip**: The `clangd-lsp` plugin (if enabled) provides powerful code intelligence for C/C++
files. When dealing with complex code (deep inheritance, heavy template usage, large files
with many symbols), prefer LSP operations over manual code reading:

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
  - When `#include` can't be found under `source_root`, search under the `project` path from UT_RULES.md
- **Link dependencies**: list of `.cc` files that must be compiled together, and any libraries
- **CMake entry**: what to add to `CMakeLists.txt`

### Step 3: Write Comprehensive Tests

Target >90% line coverage. Structure:

```cpp
#include <gtest/gtest.h>
// ALL system/STL/third-party includes BEFORE the #define block
#include <string>
#include <vector>

#define private public
#define protected public
// Dependency headers BEFORE the file under test (resolves forward declarations)
#include "Dependency.h"
#include "FileUnderTest.h"
#undef private
#undef protected

using namespace the_namespace;

// --- Construction ---
TEST(ClassNameTest, DefaultConstructor) {
  ClassName obj;
  EXPECT_EQ(obj.field_, default_value);  // direct member access OK
}

TEST(ClassNameTest, ParameterizedConstructor) {
  ClassName obj(arg1, arg2);
  EXPECT_EQ(obj.field_, arg1);
}

// --- Direct Member Access ---
TEST(ClassNameTest, SetAndGetField) {
  ClassName obj;
  obj.field_ = 42;
  EXPECT_EQ(obj.field_, 42);
}

// --- Business Logic (primary coverage target) ---
TEST(ClassNameTest, MethodWithBranch_TruePath) {
  // Setup for the if-branch → action → verify
}

TEST(ClassNameTest, MethodWithBranch_FalsePath) {
  // Setup for the else-branch → action → verify
}

// --- Private Methods (accessible via #define) ---
TEST(ClassNameTest, PrivateHelperMethod) {
  ClassName obj;
  auto result = obj.private_helper(input);  // direct call OK
  EXPECT_EQ(result, expected);
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

1. Business logic methods in `.cc` are where uncovered lines live — prioritize over getters/setters
2. Hit every branch: write a separate test for each path through `if/else` and `switch`
3. Error paths: if there is a `LOG_ERROR` or early return, write a test that triggers it
4. Templates: instantiate with at least 2 concrete types
5. Use `#define private public` to access all members — test private methods and state directly when it improves coverage

### Step 4: Integrate with CMake

Read the existing `CMakeLists.txt` in the test directory and **match its style exactly**.

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

Using project macros (if defined in the project's CMake setup):
```cmake
ut_add_module_sources(
  NAME mymodule_sources
  SOURCES ${SRC_ROOT}/MyClass.cc
  INCLUDES ${INCLUDE_DIRS}
  LIBS glog
)

ut_add_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  DEPENDS mymodule_sources
  INCLUDES ${INCLUDE_DIRS}
)
```

If the source is compiled into an OBJECT library or static lib target, link against that
instead of re-listing `.cc` files.

Using aggregator target (PREFERRED when available — check for `ut_add_simple_test`
in cmake/UTHelpers.cmake or CMakeLists.txt):
```cmake
# Header-only or prebuilt-only — no manual includes or libs needed
ut_add_simple_test(NAME FileName_ut SOURCES FileName_ut.cc)

# With coverage instrumentation for specific source files
ut_add_simple_test(
  NAME FileName_ut
  SOURCES FileName_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/FileName.cc
)

# With extra compile options for coverage sources (e.g., OpenMP, force-includes)
ut_add_simple_test(
  NAME FileName_ut
  SOURCES FileName_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/FileName.cc
  COVERAGE_COMPILE_OPTIONS -fopenmp -include ${SRC_ROOT}/util/usage.hh
)
```

**Coverage strategy for implementation files**: Always provide `COVERAGE_SOURCES` pointing
to the `.cc` file under test. Without `COVERAGE_SOURCES`, the test will exercise the code
through the prebuilt `.a` (no instrumentation → 0% coverage for that file). Example:

```cmake
ut_add_simple_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyClass.cc   # ← required for real coverage
)
```

Priority: always use `ut_add_simple_test` if defined. Fall back to `ut_add_test`
or manual patterns only if the aggregator target is not set up.

### Step 5: Build and Verify

```bash
cd <test_build_dir> && cmake .. && make -j$(nproc) && ctest --output-on-failure
```

ALL tests must pass — not just new ones. Fix errors before moving on.

### Step 6: Check Coverage per File

```bash
cd <test_root> && bash coverage.sh
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
| Segfault / ASAN null-deref | Dereferencing uninitialized pointer | Construct proper objects; avoid passing nullptr unless testing null handling |
| `cannot instantiate abstract class` | Class has pure virtual methods | Create a minimal concrete subclass in the test file for testing |
| Test links but coverage is 0% | Coverage instrumentation not enabled at build time | Ensure test was built with `ENABLE_COVERAGE=ON` |
| `#include "X.h"` not found | Header outside source_root | Search under `project` path: `find <project> -name "X.h"` |
| Test hangs indefinitely / parent waits forever | `EXPECT_EXIT` forks a child process; the child code uses OpenMP or another threading library internally — the child's thread pool is absent after fork, causing it to deadlock on any parallel section | **Do NOT use `EXPECT_EXIT` for code that uses OpenMP or fork-unsafe threading.** If the code calls `exit()` on failure AND uses OpenMP, these paths are fundamentally uncoverable via `EXPECT_EXIT`. Document as `[MaxCov]` with the reason: "calls exit() on divergence; EXPECT_EXIT incompatible with OpenMP fork-safety." |
| Test binary exits with code 1 unexpectedly | Code path calls `exit(1)` directly (e.g., numerical divergence, fatal initialization failure). This terminates the entire test binary, failing all subsequent tests. | Wrap in `EXPECT_EXIT` ONLY if OpenMP/threading is not involved. Otherwise, avoid triggering the divergence path (constrain test inputs so the path is not reached), and document remaining uncovered lines as `[MaxCov]`. |

---

## Private Access Pattern

Use `#define private public` / `#define protected public` to access all class members in tests:

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
- ALL system/STL/third-party headers (`<string>`, `<vector>`, `<gtest/gtest.h>`, etc.) MUST
  come **before** `#define private public` — otherwise STL internals break
- `#undef` after project includes restores normal semantics for the rest of the file
- More portable than `-fno-access-control` — works with both GCC and Clang
- `-fno-access-control` remains valid as an alternative via `COVERAGE_COMPILE_OPTIONS`

This pattern enables:
- Direct member field access (`obj.field_`) without getters
- Calling private/protected methods directly
- Testing internal state that would otherwise require complex public-API workarounds

---

## Source Compilation Principle

**If the project's main build succeeds, all source files under `project` root can be
compiled in the UT build environment using the same include paths.**

This has two corollaries:

1. **Every `#include` is resolvable** — When an agent cannot find a header:
   - Do NOT mark the file as [BuildFail]
   - Search under the `project` path: `find <project> -name "HeaderName.h" -type f`
   - Add the header's directory to `target_include_directories`

2. **Every `.cc` file can be recompiled from source** — Prefer compiling source files
   directly over the prebuilt library strategy. Prebuilt = 0% coverage; source = real data.
   Only use prebuilt when an external SDK or binary-only component is genuinely unavailable.

**Do NOT give up on a file because it seems hard to compile.** Read the error, find the
missing include or symbol, add it. The project already proves the code compiles.

---

## Dual Build Modes

- **Fast mode** (`run_tests.sh`): Links prebuilt `.a` via IMPORTED targets. No instrumentation.
  Build in seconds. Use for rapid iteration.
- **Coverage mode** (`coverage.sh`): Passes `-DENABLE_COVERAGE=ON`. `ut_add_module_sources`
  compiles from source as OBJECT libraries with gcov flags. `ut_add_simple_test` creates
  per-test `_cov` OBJECT libraries for targeted instrumentation. Build in minutes.

---

## Build System Simplification

### When to Use an Aggregator Target

Consider the aggregator pattern when:
- The project has **>5 test targets** sharing the same include directories
- There are **>5 shared include directories** across tests
- Tests link against **prebuilt `.a` files** from the project's build output
- **OpenMP detection** or other `find_package()` calls are duplicated across targets
- The same `target_include_directories` / `target_link_libraries` blocks are copy-pasted

### INTERFACE Aggregator Target

Create a single INTERFACE library that bundles all shared dependencies:

```cmake
# In module_config.cmake
set(PROJECT_UT_DEPS_TARGET myproject_ut_deps)
add_library(${PROJECT_UT_DEPS_TARGET} INTERFACE)

target_include_directories(${PROJECT_UT_DEPS_TARGET} INTERFACE
  ${SRC_ROOT}
  ${SRC_ROOT}/include
  # ... all shared include dirs
)

target_link_libraries(${PROJECT_UT_DEPS_TARGET} INTERFACE
  -Wl,--start-group
  ${BUILD_LIB_DIR}/libmodule_a.a
  ${BUILD_LIB_DIR}/libmodule_b.a
  -Wl,--end-group
  -Wl,--allow-multiple-definition
  -Wl,--unresolved-symbols=ignore-all
  glog pthread
)
```

Every test links against this single target — no per-test include/lib boilerplate.

### `ut_add_simple_test` Macro

Zero-config test registration:

```cmake
ut_add_simple_test(NAME Foo_ut SOURCES Foo_ut.cc)
```

Optional parameters:
- **`COVERAGE_SOURCES`** — `.cc` files to recompile with coverage instrumentation. These are
  compiled as an OBJECT library (`${NAME}_cov`) when `ENABLE_COVERAGE=ON`, linked before
  prebuilt `.a` files so instrumented symbols take precedence.
- **`COVERAGE_COMPILE_OPTIONS`** — extra flags for the coverage OBJECT library (e.g., `-fopenmp`,
  `-include header.hh`).

### Coverage OBJECT Library Pattern

When `COVERAGE_SOURCES` is provided and `ENABLE_COVERAGE=ON`:
1. A `${NAME}_cov` OBJECT library is created from the coverage sources
2. Coverage compile flags are applied to this OBJECT lib
3. The OBJECT lib is linked into the test executable **before** prebuilt `.a` files
4. The linker's `--allow-multiple-definition` flag lets the instrumented symbols win over
   the prebuilt (non-instrumented) versions

This gives real line coverage for specific source files without recompiling the entire project.

### Linker Flag Reference

| Flag | Purpose |
|------|---------|
| `--start-group` / `--end-group` | Resolve circular dependencies between `.a` files |
| `--allow-multiple-definition` | Let coverage OBJECT symbols coexist with prebuilt `.a` symbols |
| `--unresolved-symbols=ignore-all` | Partial linking when transitive dependencies are unavailable |

### Recognizing Verbose CMake

Signs that the build system would benefit from an aggregator:
- Same `target_include_directories` block repeated across **>3 targets**
- Same `target_link_libraries` list copy-pasted in multiple places
- Duplicated `find_package(OpenMP)` or similar detection blocks
- Per-test `--unresolved-symbols=ignore-all` flags

When spotted, record in `UT_RULES.md ## Project Gotchas`:
```
- [CMakeVerbose] <directory> — consider aggregator target
```

---

## Coverage Operations

### TODO.md format (produced by gen_todo.sh)

TODO.md is organized as directory sections. Entry formats:
- `- [x] Foo.cc (94% - covered)` — at ≥90%, skip
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` — below 90%, has uncovered line count
- `- [ ] Baz.cc (0%, ? uncov - no tests)` — no instrumented lines captured yet
- `- [x] Qux.cc (n/a — **[NoCode]** pure enum definitions)` — no executable code, nothing to cover
- `- [ ] Big.cc (29%, 512 uncov — **[MaxCov 29%]** reason)` — at coverage ceiling, excluded
- `- [x] Hdr.hh (n/a — **[DeclOnly]** reason)` — pure declarations, no inline bodies, gcov has nothing to instrument
- `- [ ] Hdr.hh (45%, 12 uncov — **[DeclOnly]** reason)` — has inline bodies, coverable (total>0 from gcov)

The uncovered line count is the primary input for **complex-first** scoring.

Note: `.hh`/`.h` files with no label and no data are silently omitted from TODO.md.
`[DeclOnly]` headers with `total=0` (no inline bodies) are marked `[x] n/a` — gcov has nothing to instrument.
`[DeclOnly]` headers with `total>0` (has inline bodies) are shown as `[ ]` and ARE coverable.
`[NoCode]` `.cc` files are marked `[x] n/a`.

### Regenerate coverage + TODO.md
```bash
cd <test_root> && bash coverage.sh       # build, run tests, generate coverage_filtered.info + TODO.md
cd <test_root> && bash gen_todo.sh       # re-parse existing coverage data only (faster)
```

### When to use which script

| Script | Cost | When to use |
|--------|------|-------------|
| `coverage.sh` | Minutes (full rebuild) | After writing new tests or changing source code |
| `gen_todo.sh <MODULE> <SOURCE_ROOT>` | Seconds (re-parse) | Refresh TODO.md from existing coverage data (e.g., after updating UT_RULES.md labels) |
| `run_tests.sh` | Seconds (fast build) | Quick build-and-test iteration without coverage |

Note: `coverage.sh` calls `gen_todo.sh` at the end — no need to run both.

### Check a specific file's coverage
```bash
lcov --list <test_root>/build_cov/coverage_filtered.info | grep "FileName"
```

### View uncovered lines
Open `<test_root>/coverage_report/<path>/FileName.cc.gcov.html` in a browser, or:
```bash
gcov -b -c FileName.cc  # shows branch coverage in the terminal
```

### genhtml --prefix

Use `genhtml --prefix <SOURCE_ROOT>` to strip the source root path, producing a report
directory that mirrors the source tree structure for easy navigation.

---

## C++ Test Patterns

### Data classes (structs/classes with getters/setters)

Write one test per field: set + get. Fast to write, guarantees baseline coverage.
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
Or write explicit tests with two types if `TYPED_TEST` is overkill.

### Classes with file I/O

Use `std::tmpnam` or create files under `/tmp/`:
```cpp
TEST(FileReaderTest, ReadsCorrectly) {
  auto tmp = std::string(std::tmpnam(nullptr));
  { std::ofstream f(tmp); f << "test content\n"; }
  FileReader reader(tmp);
  EXPECT_EQ(reader.read_line(), "test content");
  std::remove(tmp.c_str());
}
```

### Classes with cross-module or singleton dependencies

When a file calls into a singleton or context object from another module, that module's
source is also under `project` root — it is NOT an unavailable external dependency.

**Escalation order (try each in sequence; stop when ≥90% coverage is reached):**

1. **Search existing fixtures first**: Look in `_ut.cc` files in the same module directory
   for MinimalWrapper / test-fixture classes that already set up the required context.
   Reuse them to avoid duplicating setup logic.

2. **Write a singleton stub** (Singleton Stub Pattern — see below): If the class calls
   `AnotherModule::getInst().someMethod()`, create a stub that overrides `getInst()` and
   `someMethod()` with lightweight no-op or hardcoded implementations. Use
   `--allow-multiple-definition` so the stub wins over the prebuilt `.a` version.

3. **Compile the dependency from source**: If stubbing is insufficient, add the dependency's
   `.cc` files to `COVERAGE_SOURCES` or `target_sources` — they compile cleanly since the
   project proves they do. This gives real coverage for the dependency too.

4. **Test pure-logic methods standalone**: Static methods, pure-math utilities, enum
   accessors, and config getters do not need system context — test those first to achieve
   partial coverage before tackling context-dependent paths.

5. **Prebuilt library** (LAST RESORT — yields 0% coverage): Only when the dependency
   requires a binary-only external SDK that truly cannot be compiled in the test environment.
   See "Prebuilt Library Strategy" below.

**Key insight**: Cross-module singletons, database wrappers, and context managers are
ordinary C++ classes with source under `project`. They are NOT unresolvable system deps.
Always try stubs or source compilation before reaching for prebuilt libraries.

### Singleton Stub Pattern

When the file under test calls a singleton that requires full system initialization,
create a stub file that provides lightweight override implementations.

**Structure** (`<test_root>/<module>/stubs/Singleton_stub.cc`):
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
// Return values chosen so the test can reach the branches you want to cover.
bool SomeSingleton::doOperation(Item* /*item*/) { return true; }
double SomeSingleton::queryValue(int /*id*/) { return 0.0; }

}  // namespace project
```

**CMake integration** (with aggregator target):
```cmake
ut_add_simple_test(
  NAME MyFile_ut
  SOURCES MyFile_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyFile.cc
)
# Add stub — overrides singleton methods from the prebuilt .a
target_sources(MyFile_ut PRIVATE
  ${CMAKE_CURRENT_SOURCE_DIR}/stubs/Singleton_stub.cc
)
```

The `--allow-multiple-definition` flag in the aggregator target ensures the stub's
symbol definitions take precedence over the prebuilt `.a` versions.

**Choosing stub return values**: To cover both branches of `if (singleton.doOp())`,
you may need two test targets with different stub behaviors — or use a global flag:
```cpp
static bool g_stub_result = true;
bool SomeSingleton::doOperation(Item*) { return g_stub_result; }
```
Then set `g_stub_result = false` in a test before the call.

**TearDown**: Always call `SomeSingleton::destroyInst()` in fixture `TearDown()` to
prevent singleton state from leaking between tests.

### Fixture segfault: trace the initialization chain

When a test segfaults with a null-pointer dereference, **do not assume the file is
system-dependent or [BuildFail]**. Segfaults in fixtures almost always mean a missing
initialization step in `SetUp()`.

**Debugging process:**
1. Identify the null pointer from the crash address or ASAN output
2. Find what field was null (e.g., `_manager` in `obj->_manager->method()`)
3. Search the source for the method that populates `_manager` (e.g., `initManager()`)
4. Check if `SetUp()` calls that initialization — if not, add it

```cpp
void SetUp() override {
  obj.initContextA();       // REQUIRED: populates _context
  obj.initManager();        // REQUIRED: populates _manager (was missing, caused segfault)
  // Now obj.runOperation() no longer crashes
}
void TearDown() override {
  obj.destroyManager();
  obj.destroyContextA();    // Reverse order
}
```

**Key principle**: Read the source file's own initialization sequence (its constructor
or `init*()` methods) to discover which sub-initializations are required. The project's
integration tests or main entry point often show the correct sequence.

### Prebuilt Library Strategy (Last Resort)

**Use this ONLY when source recompilation is genuinely impossible** — for example, when
the dependency requires a binary-only external SDK, a system service (OS-level daemon,
hardware driver), or a component whose source is not available under `project` root.

For all dependencies whose source IS under `project` root (which is almost always the
case for same-project modules), prefer the Singleton Stub Pattern or direct source
compilation — both give real coverage data. The prebuilt strategy yields 0% coverage
for the linked file.

When source recompilation is truly impossible, link against prebuilt static libraries:

```cmake
# Step 1: Create a module target that wraps the prebuilt .a (no recompilation of sources).
# SOURCES "" tells the project macros to create an import-only target.
ut_add_module_sources(
  NAME mymodule_prebuilt
  SOURCES ""
  INCLUDES ${INCLUDE_DIRS}
  PREBUILT_LIB ${BUILD_LIB_DIR}/libmymodule.a
  LIBS glog
)

# Step 2: Link the test against prebuilt .a files.
# -Wl,--unresolved-symbols=ignore-all allows symbols from the external service
# (e.g., a timing engine, a database backend) to remain unresolved at link time.
ut_add_test(
  NAME MyModule_ut
  SOURCES MyModule_ut.cc
  DEPENDS mymodule_prebuilt
  INCLUDES ${INCLUDE_DIRS}
  LIBS
    ${BUILD_LIB_DIR}/libframework-core.a
    ${BUILD_LIB_DIR}/libframework-data.a
    glog pthread
    -Wl,--unresolved-symbols=ignore-all
)
```

**Trade-off**: Prebuilt binaries are not instrumented, so the file's coverage will be 0%.
This is a fallback only — if source compilation becomes feasible later, switch to
`COVERAGE_SOURCES` to get real coverage. Record as `[MaxCov 0%]` in
`UT_RULES.md ## Max Coverage Files` only after confirming source compilation is impossible.

**With an aggregator target**: Prebuilt library linking is handled automatically by the
aggregator's `target_link_libraries`. No per-test `--unresolved-symbols=ignore-all` is needed.
Use `COVERAGE_SOURCES` in `ut_add_simple_test` to get real coverage for specific files while
the aggregator provides all other symbols.

---

## File Difficulty Classification

| Difficulty | Criteria |
|-----------|---------|
| **Easy** | Header-only (no `.cc`), or `.cc` <50 lines, few internal includes |
| **Medium** | Has `.cc`, 50–150 lines, <5 internal includes, no system-level deps |
| **Hard** | Has `.cc`, >150 lines, many internal includes, some system deps but mockable |
| **System-Dependent** | Requires database/full-system context in constructor, or external services not available in the test environment |

**Complexity score for complex-first:**
- Has `.cc`: +2
- `.cc` >100 lines: +1
- Uncovered lines above median for the module: +1
- Internal `#include`s >6: +1 (capped at +2 total for this factor)
