# UT Pilot

Automated C/C++ unit test writer and coverage driver for [Claude Code](https://docs.anthropic.com/en/docs/claude-code).

UT Pilot analyzes your source code, writes comprehensive Google Test (or Catch2) tests, integrates them into CMake, and iteratively drives line coverage toward >90%.

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

## Quick Start

After installing, open Claude Code in any C/C++ project:

```
> /ut-pilot:init
```

That's it. UT Pilot will scan your project, set up the test infrastructure, and write your first test.

## Usage

### Step 1: Initialize (first time only)

```
> /ut-pilot:init
```

Bootstraps test infrastructure: creates test directory, CMakeLists.txt, coverage scripts, and a sample test. Only needed once per project.

### Step 2: Check what needs testing

```
> /ut-pilot:status
```

Shows a coverage summary: how many files are tested, which directories need work, and what to tackle next.

### Step 3: Write tests

Pick one of these approaches:

```bash
# Target a specific directory
> /ut-pilot:target_src src/core/parser/

# Or let UT Pilot auto-pick the easiest uncovered files
> /ut-pilot:continue
```

UT Pilot will:
1. Find uncovered source files
2. Read and understand each file's API
3. Write comprehensive unit tests
4. Add CMake build entries
5. Build and verify all tests pass
6. Report coverage results

### Step 4: Repeat

Just keep running `/ut-pilot:continue` to steadily increase coverage. Each run picks the next batch of files, writes tests, and reports progress.

```
> /ut-pilot:continue
> /ut-pilot:continue
> /ut-pilot:continue
...
```

## Commands Reference

| Command | Description |
|---------|-------------|
| `/ut-pilot:init` | Bootstrap UT infrastructure for a new project |
| `/ut-pilot:status` | Show coverage summary and next targets |
| `/ut-pilot:target_src <path>` | Write tests for files in a specific directory or file |
| `/ut-pilot:continue` | Auto-pick next uncovered files and write tests |

## What It Does

1. **Discovers** your project layout (CMake, build system, existing tests, frameworks)
2. **Analyzes** source files to understand APIs, dependencies, and test priorities
3. **Writes** comprehensive unit tests targeting >90% line coverage
4. **Integrates** tests into your build system (CMakeLists.txt)
5. **Verifies** all tests pass before moving on
6. **Measures** coverage and reports progress
7. **Prioritizes** easy wins first (header-only, data classes) then harder files

## Features

- Works with **Google Test**, **Catch2**, and other C++ test frameworks
- Supports **CMake**, with build system detection
- Generates **lcov/gcov** coverage reports
- **Smart prioritization**: easy files first, skips files needing complex mocking
- **Incremental**: run `/ut-pilot:continue` repeatedly to steadily increase coverage
- Handles common C++ gotchas: forward declarations, circular includes, template instantiation, logging dependencies

## Requirements

- C/C++ project with CMake
- Google Test or Catch2 installed (or fetchable)
- lcov + gcov for coverage (optional but recommended)
- Claude Code CLI

## How It Works

```
Read source -> Understand API -> Write tests -> Update CMake -> Build -> Verify -> Measure coverage
                                                                           |
                                                                    Tests fail? Fix and retry
                                                                           |
                                                                    Coverage <90%? Add more tests
```

For each source file, UT Pilot:
1. Reads both `.h`/`.hh` and `.cc`/`.cpp` files
2. Identifies all public methods, constructors, enums
3. Plans test cases covering constructors, getters/setters, business logic, edge cases, and error paths
4. Writes the test file mirroring your source directory structure
5. Adds CMake entries matching your project's existing patterns
6. Builds and runs ALL tests (not just the new one)
7. Checks coverage and adds more tests if needed

## Typical Workflow Example

```
$ claude

> /ut-pilot:init
Setting up test infrastructure for MyProject...
Created tests/ut/CMakeLists.txt
Created tests/ut/run_tests.sh
Created tests/ut/coverage.sh
Created tests/ut/MyModule/Parser_test.cc (sample test)
Built and verified: 1/1 tests passing.

> /ut-pilot:status
MyProject Coverage Status:
  Total source files: 47
  Tested (>90%):      1
  Needs tests:        46

  Easy (header-only):  12 files
  Medium (with .cc):   28 files
  Skip (deep deps):     6 files

Next targets: Config.h, Types.h, Token.h (header-only, no dependencies)

> /ut-pilot:continue
Writing tests for 5 files...
  [OK] Config.h       -> Config_test.cc       (100%)
  [OK] Types.h        -> Types_test.cc        (100%)
  [OK] Token.h        -> Token_test.cc        (95%)
  [OK] Node.h         -> Node_test.cc         (98%)
  [OK] Error.h        -> Error_test.cc        (100%)

All 6 tests passing. Coverage: 6/47 files at >90%.
Next targets: Parser.cc, Lexer.cc, AST.cc

> /ut-pilot:continue
...
```

## License

MIT
