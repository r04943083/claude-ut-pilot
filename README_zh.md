# UT Pilot

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Language: C/C++](https://img.shields.io/badge/Language-C%2FC%2B%2B-blue.svg)
![Coverage Target: >90%](https://img.shields.io/badge/Coverage%20Target-%3E90%25-brightgreen.svg)

面向 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的 C/C++ 自动化单元测试编写与覆盖率驱动工具。

UT Pilot 分析您的源代码，编写全面的 Google Test（或 Catch2）测试，将其集成到 CMake 构建系统，并迭代地将行覆盖率推向 >90%。它处理最困难的部分：私有成员访问、单例桩（Singleton Stub）、初始化链追踪、OpenMP 安全测试设计，以及快速/覆盖率双重构建模式。

---

## 功能特性

- 支持 **Google Test**、**Catch2** 等 C++ 测试框架
- 支持 **CMake**，自动集成构建系统
- 生成 **lcov/gcov** HTML 覆盖率报告
- **文件分类系统**：[NoCode]、[DeclOnly]、[BuildFail]、[MaxCov] 标签追踪哪些文件可以/不可以被覆盖
- **双重构建模式**：快速迭代构建（秒级）与完整覆盖率插桩构建（分钟级）
- **智能优先级**：复杂度优先评分，优先处理影响最大的未覆盖文件
- **增量式**：重复运行 `/ut-pilot:continue` 稳步提升覆盖率
- **CMake 聚合器模式**：`ut_add_simple_test` 宏消除每个测试的重复样板代码
- **单例桩模式（Singleton Stub Pattern）**：无需完整系统初始化，即可为跨模块依赖提供轻量级覆盖
- **初始化链追踪**：诊断并修复 fixture 段错误，而非直接标记为不可测
- **OpenMP 感知**：记录使用 `exit()` 的多线程代码的 fork 安全限制

---

## 安装

### 从 Claude Code 插件市场安装

```bash
# 第一步：添加插件源
/plugin marketplace add r04943083/claude-ut-pilot

# 第二步：安装 skill
/plugin install ut-pilot@claude-ut-pilot
```

### 从 GitHub 手动安装

```bash
claude install-skill https://github.com/r04943083/claude-ut-pilot
```

### 本地安装

```bash
git clone https://github.com/r04943083/claude-ut-pilot.git
claude install-skill ./claude-ut-pilot
```

---

## 快速开始

安装完成后，在任意 C/C++ 项目中打开 Claude Code：

```
> /ut-pilot:init
```

UT Pilot 将扫描您的项目，搭建测试基础设施，并编写第一个测试。

### 第一步：初始化（仅首次需要）

```
> /ut-pilot:init
```

初始化测试基础设施：创建测试目录、`CMakeLists.txt`、`run_tests.sh`、`coverage.sh`、`gen_todo.sh`、`UT_RULES.md`、`TODO.md` 以及一个示例测试。每个项目只需执行一次。

### 第二步：查看待测状态

```
> /ut-pilot:status
```

显示覆盖率摘要：已测试文件数量、哪些目录需要补充测试，以及按影响排序的下一批目标文件。

### 第三步：编写测试

```bash
# 针对特定目录或文件
> /ut-pilot:path src/core/parser/

# 或让 UT Pilot 自动挑选影响最大的未覆盖文件
> /ut-pilot:continue
```

UT Pilot 将：
1. 查找未覆盖的源文件
2. 读取并理解每个文件的 API
3. 编写全面的单元测试
4. 添加 CMake 构建条目
5. 构建并验证所有测试通过
6. 报告每个文件的覆盖率结果

### 第四步：重复执行

```
> /ut-pilot:continue
> /ut-pilot:continue
> /ut-pilot:continue
...
```

或全自动运行，直到所有文件达到 >90%：

```
> /ut-pilot:auto
```

---

## 架构

### 插件结构

```
plugins/ut-pilot/
├── .claude-plugin/
│   └── plugin.json          # 插件元数据（名称、描述、作者）
├── commands/                # 每个用户命令对应一个 .md 文件
│   ├── init.md              # /ut-pilot:init — 初始化基础设施
│   ├── status.md            # /ut-pilot:status — 显示覆盖率摘要
│   ├── path.md              # /ut-pilot:path — 聚焦特定目录或文件
│   ├── continue.md          # /ut-pilot:continue — 编写下一批测试
│   └── auto.md              # /ut-pilot:auto — 循环执行直到完成
└── skills/ut-pilot/
    └── SKILL.md             # 所有 agent 共享的技术参考文档
```

**命令与技能的分离**：每个命令 `.md` 文件定义一个用户命令的高层行为（做什么、按什么顺序）。`SKILL.md` 是共享技术参考——记录文件分类规则、CMake 模式、C++ 测试技术和错误修复方案。并行编写测试时派生的 agent 会加载 `SKILL.md` 作为上下文，从而一致地应用这些模式，无需重新发现。

### `test_root` 下生成的关键构件

| 构件 | 描述 |
|------|------|
| `UT_RULES.md` | 项目专属规则：命名约定、[NoCode]/[BuildFail]/[MaxCov] 标签、注意事项 |
| `TODO.md` | 每个文件的覆盖率状态与未覆盖行数；驱动 continue/auto 循环 |
| `CMakeLists.txt` | 测试构建系统；使用 `ut_add_simple_test` 聚合器宏 |
| `run_tests.sh` | 快速构建+运行（链接预编译 `.a`；秒级完成） |
| `coverage.sh` | 覆盖率构建+运行+报告生成（从源码重新编译并插桩 gcov；需数分钟） |
| `gen_todo.sh` | 重新解析已有覆盖率数据以刷新 `TODO.md`，无需重新构建 |
| `cmake/UTHelpers.cmake` | 定义 `ut_add_simple_test`、`ut_add_test`、`ut_add_module_sources` 宏 |
| `cmake/module_config.cmake` | INTERFACE 聚合器目标，打包所有共享头文件目录和预编译 `.a` 链接 |
| `build/` | 快速构建产物 |
| `build_cov/` | 覆盖率构建产物 + `.gcda`/`.gcno` 文件 |
| `coverage_report/` | genhtml 生成的 HTML 报告 |

### 双重构建模式

| 模式 | 脚本 | 用途 | 构建时间 |
|------|------|------|---------|
| **快速模式** | `run_tests.sh` | 通过预编译 `.a` 库构建并运行测试——无插桩。用于编写测试时的快速迭代。 | 秒级 |
| **覆盖率模式** | `coverage.sh` | 传入 `-DENABLE_COVERAGE=ON`，将 `COVERAGE_SOURCES` 中的源文件以 gcov 标志重新编译。生成 HTML 报告。 | 分钟级 |

`gen_todo.sh` 是一个独立的轻量级脚本，重新解析已有的 `.info` 覆盖率数据并刷新 `TODO.md`，无需任何重新构建。在 `UT_RULES.md` 中更新标签后运行它，可立即看到变更效果。

### 文件分类系统

在编写测试前，每个源文件都会被分类，以确定正确的处理方式：

| 标签 | 含义 | 处理方式 |
|------|------|---------|
| `[NoCode]` | 无可执行代码（方法体全部注释、空类体、纯枚举、仅宏/typedef/include） | 编写一个编译验证测试（`SUCCEED()`），记录到 `UT_RULES.md` |
| `[DeclOnly]` | 仅含声明的头文件——无内联方法体，无数据成员初始值 | 编写实例化/构造测试。gcov 将显示 0%（无可插桩内容），但测试仍可验证编译期正确性 |
| `[BuildFail]` | 经过 ≥3 次修复尝试（包括预编译库方案）后编译或链接仍失败 | 在 `UT_RULES.md` 中记录错误摘要，在 continue/auto 循环中跳过 |
| `[MaxCov]` | 编译运行正常，但因架构限制无法进一步提升覆盖率 | 在 `UT_RULES.md ## Max Coverage Files` 中记录原因和已达覆盖率，在 continue/auto 循环中跳过 |

`[NoCode]` 和 `[BuildFail]` 与 `[MaxCov]` 互斥。能编译但有 `exit()` 路径的文件不能同时标注 `[BuildFail]`。

### 覆盖率优先级策略

continue/auto 循环使用**复杂度优先**评分公式选取文件：

| 因素 | 分值 |
|------|------|
| 有对应 `.cc` 文件 | +2 |
| `.cc` 超过 100 行 | +1 |
| 未覆盖行数高于模块中位数 | +1 |
| 内部 `#include` 超过 6 个 | +1（总分上限 +2） |

`TODO.md` 中的未覆盖行数是主要输入。未覆盖行数更多、复杂度更高的文件优先处理，以最大化每次会话的覆盖率收益。

`/ut-pilot:path` 命令覆盖优先级策略，将所有工作聚焦在特定目录或文件上。此设置以 `current_focus` 形式持久化到 `UT_RULES.md`。

---

## 命令参考

### `/ut-pilot:init`

为新的 C/C++ 项目初始化测试基础设施。每个项目执行一次。

- 发现项目布局：CMake 结构、源码根目录、构建系统、测试框架
- 创建带有 `CMakeLists.txt`、shell 脚本和 CMake 辅助文件的 `test_root/`
- 生成包含检测到的项目约定和命名规则的 `UT_RULES.md`
- 通过扫描源文件生成初始 `TODO.md`
- 编写示例测试以端到端验证构建可用

### `/ut-pilot:status`

显示当前测试覆盖率状态和下一批目标。

- 读取 `TODO.md` 汇总每个目录和整体覆盖率
- 列出达到 ≥90% 的文件数、需要测试的文件数以及下一批目标
- 单独报告 [NoCode]、[DeclOnly]、[BuildFail]、[MaxCov] 文件数量

### `/ut-pilot:path <路径>`

设置下次 continue 运行的聚焦目录（或文件）。

- 将 `current_focus = <路径>` 保存到 `UT_RULES.md`
- 后续的 `/ut-pilot:continue` 运行只针对该路径
- 适用于在恢复全局覆盖率工作前深入处理特定模块

### `/ut-pilot:continue`

自动挑选下一批未覆盖文件并编写测试。

- 按复杂度优先评分读取排序后的 `TODO.md`
- 跳过标有 [NoCode]、[BuildFail]、[MaxCov] 或已达 ≥90% 的文件
- 对每个挑选的文件：读取源码、分类、编写测试、更新 CMake、构建、验证覆盖率
- 每个文件完成后用新的覆盖率数据更新 `TODO.md`

### `/ut-pilot:auto`

自动循环执行 `/ut-pilot:continue`，直到所有文件达到 >90% 覆盖率或无法再取得进展。等效于在循环中运行 `continue`。完成时报告最终摘要。

---

## CMake 集成

### 基础模式（`ut_add_test`）

用于有明确源文件和依赖列表的测试：

```cmake
ut_add_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  DEPENDS mymodule_sources
  INCLUDES ${INCLUDE_DIRS}
)
```

### 聚合器模式（`ut_add_simple_test`）

零配置测试注册——聚合器目标设置好后的首选方式：

```cmake
# 纯头文件或纯逻辑——无需额外源文件
ut_add_simple_test(NAME FileName_ut SOURCES FileName_ut.cc)
```

该宏自动链接共享的 INTERFACE 聚合器目标（`myproject_ut_deps`），后者打包了项目所有头文件目录和预编译 `.a` 文件。

### `COVERAGE_SOURCES`——真实覆盖率的必要条件

不提供 `COVERAGE_SOURCES` 时，测试通过预编译 `.a` 执行代码——该文件未经插桩，**覆盖率为 0%**。

```cmake
ut_add_simple_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyClass.cc   # 真实覆盖率的必要条件
)
```

当 `ENABLE_COVERAGE=ON`（即执行 `coverage.sh` 时），会从 `MyClass.cc` 创建带 gcov 标志的 `MyClass_ut_cov` OBJECT 库，并在预编译 `.a` 之前链接，使插桩符号通过 `--allow-multiple-definition` 优先生效。

携带额外编译选项（如 OpenMP 或强制包含头文件）：

```cmake
ut_add_simple_test(
  NAME MyClass_ut
  SOURCES MyClass_ut.cc
  COVERAGE_SOURCES ${SRC_ROOT}/path/to/MyClass.cc
  COVERAGE_COMPILE_OPTIONS -fopenmp -include ${SRC_ROOT}/util/usage.hh
)
```

### 桩集成

单例桩作为额外的私有源文件添加到测试目标：

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

### 链接器标志参考

| 标志 | 用途 |
|------|------|
| `--start-group` / `--end-group` | 解决 `.a` 文件之间的循环依赖 |
| `--allow-multiple-definition` | 让覆盖率 OBJECT 符号覆盖预编译 `.a` 中的符号 |
| `--unresolved-symbols=ignore-all` | 传递依赖不可用时进行部分链接 |

---

## C++ 测试技术

### 私有成员访问（`#define private public`）

在测试中直接访问所有私有/受保护字段和方法：

```cpp
#include <gtest/gtest.h>
#include <string>       // 所有系统/STL 头文件必须在 #define 块之前

#define private public
#define protected public
#include "MyClass.h"    // 项目头文件在 #define 之后
#undef private
#undef protected
```

**规则：**
- 所有 `<系统>` 和第三方头文件（`<gtest/gtest.h>`、`<string>`、`<vector>`）**必须**在 `#define private public` 之前——否则 STL 内部会出错
- 项目 include 之后的 `#undef` 恢复文件其余部分的正常语义
- 比 `-fno-access-control` 更具可移植性；兼容 GCC 和 Clang

此模式允许直接访问成员字段（`obj.field_`）和调用私有方法，无需任何公共 API 迂回。

### 单例桩模式（Singleton Stub Pattern）

当被测文件调用需要完整系统初始化的单例时，在 `<test_root>/<module>/stubs/Singleton_stub.cc` 中创建轻量级桩覆盖：

```cpp
// 为被测代码调用的 SomeSingleton 方法提供轻量级覆盖。
// --allow-multiple-definition 链接器标志让此文件覆盖预编译 .a 中的版本。
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

// 只覆盖被测文件实际调用的方法。
// 返回值的选择要能让测试覆盖到所有目标分支。
bool SomeSingleton::doOperation(Item* /*item*/) { return true; }
double SomeSingleton::queryValue(int /*id*/) { return 0.0; }

}  // namespace project
```

若要覆盖 `if (singleton.doOp())` 的两个分支，在桩中使用全局标志：

```cpp
static bool g_stub_result = true;
bool SomeSingleton::doOperation(Item*) { return g_stub_result; }
// 测试中：g_stub_result = false; obj.method();  // → 覆盖 false 分支
```

始终在 fixture 的 `TearDown()` 中调用 `SomeSingleton::destroyInst()`，防止单例状态在测试间泄漏。

**关键认识**：跨模块单例是项目根目录下普通的 C++ 类，不是无法解析的系统依赖。在使用预编译库（覆盖率为 0%）之前，始终先尝试桩或直接编译源码。

### fixture 初始化链追踪

当测试因空指针解引用而段错误时，这几乎总意味着 `SetUp()` 中缺少某个初始化步骤——而不是文件系统相关或 `[BuildFail]`。

**调试过程：**
1. 从崩溃地址或 ASAN 输出中确定空指针
2. 找出哪个字段为空（例如 `obj->_manager->method()` 中的 `_manager`）
3. 在源码中搜索填充 `_manager` 的方法（例如 `initManager()`）
4. 将缺失的调用添加到 `SetUp()`，并在 `TearDown()` 中添加对应的反初始化

```cpp
void SetUp() override {
  obj.initContextA();    // 填充 _context
  obj.initManager();     // 填充 _manager——缺失此调用导致段错误
}
void TearDown() override {
  obj.destroyManager();
  obj.destroyContextA(); // 逆序
}
```

阅读源文件自身的构造函数或 `init*()` 方法，发现所需的初始化序列。项目的集成测试或主入口通常展示正确的顺序。

### OpenMP / 多线程约束

**不要对使用 OpenMP 或其他非 fork 安全线程的代码使用 `EXPECT_EXIT`。**

`EXPECT_EXIT` fork 子进程时，子进程中 OpenMP 线程池缺失。子进程中的任何并行段都会死锁，导致父进程永远等待。

如果某个代码路径在失败时调用 `exit()` 且使用了 OpenMP（例如求解器中的数值发散检测），该路径从根本上无法通过 `EXPECT_EXIT` 覆盖。将其记录为 `[MaxCov]`：

```
- **[MaxCov] NesterovPlace.cc (77%, 251 uncov)**：runNesterovPlace 数值发散时调用 exit(1)；
  EXPECT_EXIT 与 OpenMP fork 安全不兼容。可达最大覆盖率：77%。
```

---

## 设计原则

### 源码编译原则

**如果项目主构建成功，项目根目录下的每个 `.cc` 文件都可以在 UT 构建环境中使用相同的 include 路径重新编译。**

这有两个推论：

1. **每个 `#include` 都是可解析的。** 当 agent 找不到头文件时：不要将文件标记为 `[BuildFail]`。在项目路径下搜索，将目录添加到 `target_include_directories`，然后重试。

2. **每个 `.cc` 都可以从源码编译。** 优先通过 `COVERAGE_SOURCES` 直接编译源文件，而非链接预编译库。预编译 = 0% 覆盖率；源码编译 = 真实插桩数据。

不要因为文件看起来难以编译就放弃。项目本身已经证明这段代码可以编译。

### 预编译库是最后手段

预编译库策略（链接 `.a` 而不重新编译）会导致被链接文件的覆盖率为 **0%**。仅在以下情况使用：
- 依赖需要仅提供二进制的外部 SDK
- 组件源码确实不在项目根目录下
- 包括源码重新编译在内的三种修复策略均已失败

被迫使用预编译时，在 `UT_RULES.md ## Max Coverage Files` 中记录该文件，一旦源码编译可行，立即切换到 `COVERAGE_SOURCES`。

### 跨模块依赖

处理跨模块或单例依赖的升级顺序（达到 ≥90% 覆盖率时停止）：

1. **复用已有 fixture**——在同模块的 `_ut.cc` 文件中搜索已有 MinimalWrapper / 测试 fixture 类
2. **编写单例桩**——通过 `--allow-multiple-definition` 提供轻量级无操作覆盖
3. **从源码编译依赖**——将其 `.cc` 文件添加到 `COVERAGE_SOURCES`
4. **独立测试纯逻辑方法**——静态方法、纯数学工具函数和配置 getter 不需要系统上下文
5. **预编译库**（最后手段）——仅用于真正无法获得源码的二进制外部 SDK

---

## 环境要求

- 使用 CMake 的 C/C++ 项目
- 已安装的 Google Test 或 Catch2（或可通过 FetchContent 获取）
- lcov + gcov 用于覆盖率报告（可选但推荐）
- Claude Code CLI

---

## 许可证

MIT
