---
description: "Auto-pick next uncovered files and write tests to increase coverage"
---

# /ut-pilot:continue — Single Batch Test Writing

**Responsibility**: Process exactly ONE batch of files, then STOP and report.
Do not loop. Do not start the next batch automatically.

## Step 1: Load Configuration

Read `UT_RULES.md` from the test_root (search: project root, `tests/ut/`, `test/ut/`,
`tests/`, `test/`). Parse the `## Configuration` section:

- `source_root`, `test_root`, `project`, `framework`, `naming_convention`
- `parallel_agents` (default: 5)
- `files_per_batch` (default: 10)
- `strategy` (default: complex-first)
- `current_focus` (empty = global mode; non-empty = restrict to that directory)

Also read `## Project Gotchas` — pass the path to agents so they can avoid known pitfalls
and find existing fixture patterns.

If `UT_RULES.md` is not found, stop and say:
"UT_RULES.md not found. Run /ut-pilot:init first."

## Step 2: Load TODO.md

Read `TODO.md` from test_root. Format: directory sections with bullet entries:
- `- [x] Foo.cc (94% - covered)` → at ≥90%, skip
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` → actionable
- `- [ ] Baz.cc (0%, ? uncov - no tests)` → no coverage data yet

If `TODO.md` does not exist or is older than 1 hour:
- If `<test_root>/build_cov/coverage_filtered.info` exists:
  → run `cd <test_root> && bash gen_todo.sh <MODULE> <SOURCE_ROOT>` (fast, seconds — re-parses existing coverage data)
- If `coverage_filtered.info` does not exist:
  → run `cd <test_root> && bash coverage.sh` (full rebuild, minutes)

Then re-read the generated `TODO.md`.

Collect all `- [ ]` entries as the uncovered file list.

Apply `current_focus` filter if set.

Exclude files matching ANY of the following in `UT_RULES.md`:
- Recorded as `[BuildFail]` in `## Project Gotchas`
- Recorded as `[NoCode]` in `## Project Gotchas`
- Listed in `## Max Coverage Files` (already at documented coverage ceiling)

Do NOT exclude `[DeclOnly]` files — these are testable (write instantiation tests).

Do NOT exclude any other files — every remaining file must be attempted.

If no actionable files remain, report completion and stop.

## Step 3: Score and Select Files

Apply the **complex-first** scoring (or the configured strategy) to rank uncovered files:

### complex-first scoring (higher = process first):
- Has a `.cc` implementation file: **+2**
- `.cc` file has >100 lines: **+1**
- Uncovered lines count (from TODO.md) is above median: **+1**
- Number of project-internal `#include`s in the header (non-system headers): **+1 per 3 includes, max +2**

### medium-first scoring:
- Has `.cc`: **+1**
- `.cc` lines 50–150: **+2**
- Few internal includes (<3): **+1**

### simple-first scoring:
- Header-only (no `.cc`): **+3**
- Few includes: **+2**
- Many includes or complex deps: penalty

Select top `files_per_batch` files. **Do NOT auto-mark any file as [Skip].**

## Step 4: Assign Files to Agents

Divide the selected files evenly across `parallel_agents` agents (round-robin).

Each agent receives only paths and instructions — the agent reads files itself:

```
Assigned files: [list of source file paths]
Test root: <test_root>
Source root: <source_root>
Project root: <project>
UT_RULES.md path: <path to UT_RULES.md>
Test directory (for CMake patterns): <test_root>/<module>/

Your task:
1. Read assigned source files (use LSP document_symbols first if >200 lines)
2. Check existing CMakeLists.txt in the test directory for cmake patterns.
   **Look for `ut_add_simple_test` macro availability (in cmake/UTHelpers.cmake or
   CMakeLists.txt). If available, ALWAYS use it instead of manual include/lib specifications.**
3. Read UT_RULES.md for gotchas and conventions (especially ## Project Gotchas for
   existing fixture patterns)
4. Classify each file (NoCode / BuildFail / testable) — see SKILL.md.
   DeclOnly headers are testable — write instantiation tests for them.
5. Use the `#define private public` pattern for ALL test files (see SKILL.md Private Access Pattern)
6. Write tests targeting >90% line coverage per file
   - For files with system-context dependencies: search other _ut.cc files in the same
     module directory for existing fixtures/wrappers and reuse them
   - For files that cannot be recompiled from source: use the prebuilt library strategy
7. Update CMakeLists.txt using the **simplest available pattern**:
   PREFERRED: `ut_add_simple_test` if aggregator is available.
   FALLBACK: `ut_add_test` or manual.
   **If the build system is verbose (same includes/libs repeated >3 targets) and no
   aggregator exists, note in UT_RULES.md `## Project Gotchas`:
   `[CMakeVerbose] <directory> — consider aggregator target`.**
8. When #include cannot be resolved under source_root, search under project root.
   If any #include cannot be found, DO NOT mark as [BuildFail].
   The project compiles, so the header exists. Search: find <project> -name "Header.h"
9. If you discover a gotcha or useful fixture pattern, append to UT_RULES.md ## Project Gotchas.
   Do NOT add entries to ## Max Coverage Files unless a file genuinely cannot exceed the
   documented ceiling after exhausting all strategies.
Report: files created/modified and expected coverage level.
```

## Step 5: Execute Agents

If `parallel_agents` > 1: spawn N Agent calls in parallel (use the Agent tool).
If `parallel_agents` = 1: execute sequentially.

Wait for ALL agents to complete before proceeding.

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
- If still failing after 3 attempts: record in `UT_RULES.md ## Project Gotchas` as
  `[BuildFail] FileName.cc — <error summary>` (do NOT mark TODO.md — coverage.sh will
  overwrite it). Only use `[BuildFail]` for actual build failures, not runtime crashes or
  low coverage.

### 6b. Update Coverage
```bash
cd <test_root> && bash coverage.sh   # full rebuild — required here to capture new test results
```

This regenerates `TODO.md` with updated percentages.

### 6c. Check New File Coverage

For each file processed this batch with coverage <90%: read the coverage HTML report to
identify uncovered lines, add targeted tests for those branches, rebuild and recheck.
Repeat up to 2 iterations. Every file is expected to be improvable.

### 6d. Persist Gotchas

Ensure any new gotchas reported by agents are saved to `UT_RULES.md ## Project Gotchas`.

## Step 7: Report and STOP

Print a report, then STOP. Do not begin another batch.

```
## Batch Complete

| File | Before | After | Status |
|------|--------|-------|--------|
| path/to/Foo.cc | 0% | 94% | NEW |
| path/to/Bar.cc | 23% | 88% | IMPROVED (below 90%, needs more) |
| path/to/Baz.cc | 0% | 12% | STARTED (needs more tests) |

**This batch**: processed N files, M newly at >90%
**Overall**: X/Y files at >90% coverage
**Remaining**: Z files need tests

### Next Targets (complex-first)
1. module/buffer/Next1.cc — 0%, 245 uncov
2. module/placer/Next2.cc — 31%, 112 uncov
3. utility/Next3.cc — 0%, 88 uncov

Run /ut-pilot:continue for the next batch.
```

**CRITICAL**: After printing this report, stop. Do not loop. The user decides when to run
the next batch.
