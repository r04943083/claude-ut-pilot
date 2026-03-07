---
description: "Auto-pick next uncovered files and write tests to increase coverage"
---

# /ut-pilot:continue — Single Batch Test Writing

**Responsibility**: Process exactly ONE batch of files, then STOP and report. Do not loop. Do not start the next batch automatically.

## Step 1: Load Configuration

Read `UT_RULES.md` from the test_root (search: project root, `tests/ut/`, `test/ut/`, `tests/`, `test/`). Parse the `## Configuration` section:

- `source_root`, `test_root`, `framework`, `naming_convention`
- `parallel_agents` (default: 3)
- `files_per_batch` (default: 5)
- `strategy` (default: complex-first)
- `current_focus` (empty = global mode; non-empty = restrict to that directory)

Also read `## Project Gotchas` — pass these to each agent to avoid known pitfalls.

If `UT_RULES.md` is not found, stop and say: "UT_RULES.md not found. Run /ut-pilot:init first."

## Step 2: Load TODO.md

Read `TODO.md` from test_root. Parse the `## Needs Tests` table for uncovered files.

If `TODO.md` doesn't exist or is older than 1 hour, run:
```bash
cd <test_root> && bash coverage.sh
```
Then re-read the generated `TODO.md`.

Filter by `current_focus`: if non-empty, only consider files under that path.

If no uncovered files remain (or all are marked Skip), report completion and stop.

## Step 3: Score and Select Files

Apply the **complex-first** scoring (or the configured strategy) to rank uncovered files:

### complex-first scoring (higher = process first):
- Has a `.cc` implementation file: **+2**
- `.cc` file has >100 lines: **+1**
- Uncovered lines count (from TODO.md) is above median: **+1**
- Number of project-internal `#include`s in the header (non-system headers): **+1 per 3 includes, max +2**
- Depends on known-problematic classes (PlacerDB, full system context): **mark Skip, score = -99**

### medium-first scoring:
- Has `.cc`: **+1**
- `.cc` lines 50-150: **+2**
- Few internal includes (<3): **+1**

### simple-first scoring:
- Header-only (no `.cc`): **+3**
- Few includes: **+2**
- Many includes or complex deps: penalty

Select top `files_per_batch` files (excluding any already marked Skip).

If a file should be marked Skip (deep dependency, no test infrastructure support), add `[Skip]` annotation to its row in `TODO.md` and exclude it.

## Step 4: Assign Files to Agents

Divide the selected files evenly across `parallel_agents` agents (round-robin). Each agent gets 1-2 files.

Build the agent prompt for each agent. Include:
1. The source files' full content (`.h` + `.cc`) for its assigned files
2. The relevant CMakeLists.txt section from the test directory (existing patterns)
3. `UT_RULES.md` — `## Configuration` + `## Test Conventions` + `## Project Gotchas` sections
4. The Write Test Workflow from the ut-pilot SKILL.md

Agent task instruction:
```
You are a unit test writer. Your task:
1. Read the provided source file(s)
2. Write comprehensive test file(s) targeting >90% line coverage
3. Update CMakeLists.txt to build and register the new test(s)
4. Follow the naming convention: {FileName}_ut.cc (or as specified in UT_RULES.md)
5. Mirror the source directory structure under test_root
6. If you discover a gotcha (build failure pattern, include ordering issue, etc.),
   append it to the ## Project Gotchas section of UT_RULES.md

Do NOT run builds. The main process handles verification.
Report: list of files you created/modified.
```

## Step 5: Execute Agents

If `parallel_agents` > 1: spawn N Agent calls in parallel (use the Agent tool).
If `parallel_agents` = 1: execute sequentially.

Wait for ALL agents to complete before proceeding. Do not do other work while waiting.

## Step 6: Verification Loop (main process)

After all agents finish:

### 6a. Build
```bash
cd <test_root> && bash run_tests.sh
```

If build fails:
- Diagnose the error (missing include, linker error, etc.)
- Apply fix directly (do not delegate back to agents)
- Retry up to 3 times
- If still failing after 3 attempts: mark affected files as `[BuildFail]` in TODO.md, continue to coverage step with whatever passes

### 6b. Update Coverage
```bash
cd <test_root> && bash coverage.sh
```

This regenerates `TODO.md` with updated percentages.

### 6c. Check New File Coverage

For each file processed this batch: if new coverage <90%, read the coverage HTML report to identify uncovered lines, add targeted tests for uncovered branches. Rebuild and recheck (max 2 iterations).

### 6d. Persist Gotchas

If any agent reported new gotchas, ensure they are appended to `UT_RULES.md ## Project Gotchas`.

## Step 7: Report and STOP

Print a report, then STOP. Do not begin another batch. Do not call continue again.

```
## Batch Complete

| File | Before | After | Status |
|------|--------|-------|--------|
| path/to/Foo.cc | 0% | 94% | NEW |
| path/to/Bar.cc | 23% | 88% | IMPROVED (below 90%) |

**This batch**: processed N files, M newly at >90%
**Overall**: X/Y files at >90% coverage
**Remaining**: Z files need tests

### Next Targets (complex-first)
1. path/to/Next1.cc — 0%, 245 uncovered lines
2. path/to/Next2.cc — 31%, 112 uncovered lines
3. path/to/Next3.cc — 0%, 88 uncovered lines

Run /ut-pilot:continue for the next batch.
```

**CRITICAL**: After printing this report, stop. Do not loop. The user decides when to run the next batch.
