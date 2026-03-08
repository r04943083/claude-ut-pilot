---
description: "Auto-pick next uncovered files and write tests to increase coverage"
---

# /ut-pilot:continue — Single Batch Test Writing

**Responsibility**: Process exactly ONE batch of files, then STOP and report. Do not loop. Do not start the next batch automatically.

## Step 1: Load Configuration

Read `UT_RULES.md` from the test_root (search: project root, `tests/ut/`, `test/ut/`, `tests/`, `test/`). Parse the `## Configuration` section:

- `source_root`, `test_root`, `framework`, `naming_convention`
- `parallel_agents` (default: 5)
- `files_per_batch` (default: 10)
- `strategy` (default: complex-first)
- `current_focus` (empty = global mode; non-empty = restrict to that directory)

Also read `## Max Coverage Files` → build `maxcov_files` set.
Also read `## Project Gotchas` — pass the path to agents so they can avoid known pitfalls.

If `UT_RULES.md` is not found, stop and say: "UT_RULES.md not found. Run /ut-pilot:init first."

## Step 2: Load TODO.md

Read `TODO.md` from test_root. It is organized as directory sections (e.g. `## module/buffer`) with bullet entries:
- `- [x] Foo.cc (94% - covered)` → at ≥90%, skip
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` → actionable (coverage, uncovered line count)
- `- [ ] Baz.cc (0%, ? uncov - no tests)` → no coverage data yet

Collect all `- [ ]` entries as the uncovered file list.

If `TODO.md` doesn't exist or is older than 1 hour, run:
```bash
cd <test_root> && bash coverage.sh
```
Then re-read the generated `TODO.md`.

Filter by `current_focus`: if non-empty, only consider files under that path.

Exclude files in `maxcov_files` set and files marked `[BuildFail]`.

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
- `.cc` lines 50-150: **+2**
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
UT_RULES.md path: <path to UT_RULES.md>
Test directory (for CMake patterns): <test_root>/<module>/

Your task:
1. Read assigned source files (use LSP document_symbols first if >200 lines)
2. Check existing CMakeLists.txt in the test directory for cmake patterns
3. Read UT_RULES.md for gotchas and conventions
4. Write tests targeting >90% line coverage per file
5. Update CMakeLists.txt
6. If you cannot reach >90% due to system dependencies, write what IS testable,
   then add an entry to UT_RULES.md '## Max Coverage Files' with reason:
   `- FileName.cc (MaxCov: X%) — reason`
7. If you discover a gotcha, append it to UT_RULES.md '## Project Gotchas'.
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
- If still failing after 3 attempts: mark affected files as `[BuildFail]` in TODO.md, continue to coverage step with whatever passes

### 6b. Update Coverage
```bash
cd <test_root> && bash coverage.sh
```

This regenerates `TODO.md` with updated percentages.

### 6c. Check New File Coverage

For each file processed this batch: if new coverage <90%, re-read `UT_RULES.md ## Max Coverage Files` to check if an agent added a MaxCov entry for it. If not, read the coverage HTML report to identify uncovered lines and add targeted tests for uncovered branches. Rebuild and recheck (max 2 iterations).

### 6d. Persist Gotchas

Ensure any new gotchas reported by agents are saved to `UT_RULES.md ## Project Gotchas`.

## Step 7: Report and STOP

Print a report, then STOP. Do not begin another batch.

```
## Batch Complete

| File | Before | After | Status |
|------|--------|-------|--------|
| path/to/Foo.cc | 0% | 94% | NEW |
| path/to/Bar.cc | 23% | 88% | IMPROVED (below 90%) |
| path/to/Baz.cc | 0% | 0% | MaxCov: 0% — requires PlacerDB |

**This batch**: processed N files, M newly at >90%, K documented MaxCov
**Overall**: X/Y files at >90% coverage
**Remaining**: Z files need tests

### Next Targets (complex-first)
1. module/buffer/Next1.cc — 0%, 245 uncov
2. module/placer/Next2.cc — 31%, 112 uncov
3. utility/Next3.cc — 0%, 88 uncov

Run /ut-pilot:continue for the next batch.
```

**CRITICAL**: After printing this report, stop. Do not loop. The user decides when to run the next batch.
