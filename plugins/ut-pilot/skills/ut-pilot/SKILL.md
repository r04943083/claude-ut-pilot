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

## Step 0: 开始前先分类文件

写测试前必须先分类，避免错误的 [BuildFail] 标签：

### 分类 A — [NoCode]：无可执行代码
- 所有方法体都被注释掉（如 DPNode.cc, Geometry.cc）
- class 体为空（如 PostGPConfig.hh `class X {};`）
- 纯 enum 定义（如 Orient.hh）
- 纯宏/typedef 定义（如 Log.hh）

**处理**：写一个编译验证测试（`TEST(X, Compiles) { SUCCEED(); }`），
在 UT_RULES.md ## Project Gotchas 记录为 `[NoCode] FileName — reason`。
**不要写 [BuildFail]**。

### 分类 B — [DeclOnly]：仅声明的 .hh
- 只有 class 声明和方法签名，没有 inline 方法体 `{ }`
- 没有带值的数据成员初始化

**处理**：gcov 永远无法对非可执行的声明行插桩，0% 是预期行为。
不需要测试，也**不要写 [BuildFail]**。
在 UT_RULES.md ## Project Gotchas 记录为 `[DeclOnly] FileName.hh — declaration-only header`。

### 分类 C — 编译失败 vs 运行时崩溃（重要区分）

| 情况 | 分类 | 处理方式 |
|---|---|---|
| `cmake` 构建命令返回非0（编译器/链接器错误） | [BuildFail] | 修复或 3 次后记录 |
| cmake 成功但测试二进制运行时崩溃 | **不是** [BuildFail] | 用 prebuilt .a 绕开崩溃路径 |
| 文件编译正常但覆盖率达到上限 | **不是** [BuildFail] | 记录 [MaxCov] |

**你必须实际运行 cmake 才能添加 [BuildFail]。仅靠读代码推断不允许添加 [BuildFail]。**

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
If the constructor requires a database or system-context object, use this escalation order:

1. **Search existing fixtures first**: Look in `_ut.cc` files in the SAME module directory for MinimalWrapper / test-fixture classes that already set up the required dependency. Reuse them.
2. **Write a minimal stub**: If no fixture exists, write a minimal concrete subclass or struct that satisfies the required interface with no-op or hardcoded values.
3. **Test pure-logic methods standalone**: Static methods, pure-math utilities, enum accessors, and config getters do not need system context — test those first.
4. **Never give up without compiling**: The project is guaranteed to compile. Write at least one passing test for every file.

---

## Difficulty Classification

Use this to classify files for strategy scoring:

| Difficulty | Criteria |
|-----------|---------|
| **Easy** | Header-only (no `.cc`), or `.cc` <50 lines, few internal includes |
| **Medium** | Has `.cc`, 50-150 lines, <5 internal includes, no system-level deps |
| **Hard** | Has `.cc`, >150 lines, many internal includes, some system deps but mockable |
| **System-Dependent** | Requires database/full-system context in constructor, external services, or integration-level setup — find or write a minimal fixture/stub, then test through it |

Complexity score for **complex-first**:
- Has `.cc`: +2
- `.cc` >100 lines: +1
- Uncovered lines above median for the module: +1
- Internal includes >6: +1 (capped at +2 total for this)
- System-level dependency: find or write a stub/fixture (see "Classes with deep system dependencies" above)

### 利用 src 预编译库（prebuilt .a）处理系统依赖文件

当一个 .cc 文件的 include 链包含 PLAPI/IDB builder 等重型系统头文件时：

**优先策略**：不从源码重新编译，用预编译 .a 满足链接依赖：

```cmake
# 1. 用 SOURCES "" 让 UTHelpers 创建纯导入库（不重新编译源文件）
ut_add_module_sources(
  NAME mymodule_prebuilt
  SOURCES ""
  INCLUDES ${INCLUDE_DIRS}
  PREBUILT_LIB ${IEDA_BUILD_LIB}/libipl-module-xyz.a
  LIBS glog
)

# 2. 测试链接多个 prebuilt .a，用 --unresolved-symbols=ignore-all 跳过无法解析的符号
ut_add_test(
  NAME MyModule_ut
  SOURCES MyModule_ut.cc
  DEPENDS mymodule_prebuilt
  INCLUDES ${INCLUDE_DIRS}
  LIBS
    ${IEDA_BUILD_LIB}/libipl-api.a
    ${IEDA_BUILD_LIB}/libipl-source.a
    ${IEDA_BUILD_LIB}/libipl-configurator.a
    ${IEDA_BUILD_LIB}/libipl-module-topology_manager.a
    ${IEDA_BUILD_LIB}/libipl-module-grid_manager.a
    ${IEDA_BUILD_LIB}/libflute.a
    ${IEDA_BUILD_LIB}/libusage.a
    glog pthread ${_OMP_LIB}
    -Wl,--unresolved-symbols=ignore-all   # 允许链接时存在未解析符号
)
```

注意：用这个模式时，该 .cc 文件本身的覆盖率为 0%（prebuilt 不含插桩）。
这是可以接受的 —— 目标是让测试能构建和运行，然后记录 [MaxCov 0%]。
如果能从源码编译（有些文件可以），则使用 `SOURCES file.cc` 方式以获得覆盖率。

### [MaxCov] 记录规范

当文件构建正常、测试能运行，但覆盖率因架构原因无法再提升时：
1. 达到最大可达覆盖率后记录
2. 在 UT_RULES.md ## Max Coverage Files 新增条目：
   `- **[MaxCov] FileName.cc (X%, Y uncov)**: 原因说明。Maximum achievable: X%.`
3. **不要** 在 ## Project Gotchas 里写 [BuildFail]（文件是能编译的！）
4. **不要** 把 [MaxCov] 和 [BuildFail] 同时写 —— 两者互斥
