---
description: "Show current test coverage status and next targets"
---

# /ut-pilot:status — Coverage Status Report

## Step 1: Load UT_RULES.md

Read `UT_RULES.md` from the test_root (search: project root, `tests/ut/`, `test/ut/`, `tests/`, `test/`).

If not found: run a minimal project scan and report what was found (no config yet). Suggest running `/ut-pilot:init`.

Parse:
- `test_root`, `source_root`, `strategy`
- `current_focus` (may be empty)
- `files_per_batch`, `parallel_agents`

## Step 2: Load or Generate Coverage Data

Check if `TODO.md` exists in `test_root`:
- If it exists: read it directly
- If it doesn't exist: run `cd <test_root> && bash coverage.sh` to generate it, then read

Parse `TODO.md`. Format: directory sections with bullet entries:
- `- [x] Foo.cc (94% - covered)` → covered
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` → needs tests (includes uncovered line count)
- `- [ ] Baz.cc (0%, ? uncov - no tests)` → needs tests

Count: covered (≥90%), needs tests `- [ ]`.
Also read `UT_RULES.md ## Project Gotchas` and count lines starting with `[BuildFail]` for the build-failure count.
Group "Needs Tests" files by their section header (directory).

## Step 3: Print Status Report

```
## UT Coverage Status
Source root: <source_root>
Test root:   <test_root>
Strategy:    <strategy>
```

If `current_focus` is non-empty, highlight it prominently:
```
*** FOCUSED MODE: <current_focus> ***
(continue will only process files from this directory)
Run /ut-pilot:path with no args to return to global mode.
```

### Overall Summary Table

```
| Metric | Count |
|--------|-------|
| Total source files | N |
| At >90% coverage | X (XX%) |
| Need tests | Y |
| Marked [BuildFail] | W |
```

### By Directory

Group "Needs Tests" files by their immediate parent directory under source_root:

```
| Directory | Covered | Needs Tests | Skip | Top Uncovered File |
|-----------|---------|-------------|------|-------------------|
| src/algo/ | 3/8 | 4 | 1 | BigAlgo.cc (0%, 340 lines) |
| src/util/ | 12/14 | 2 | 0 | Formatter.cc (45%, 88 lines) |
```

If `current_focus` is set, mark that directory's row with `***`.

### Next Recommended Batch

Apply `strategy` scoring to all "Needs Tests" files (same logic as continue.md). Show top `files_per_batch` files:

```
### Next <files_per_batch> Targets (<strategy>)

1. src/algo/BigAlgo.cc — 0% coverage, 340 uncovered lines [complexity: high]
2. src/algo/Optimizer.cc — 12%, 180 uncovered lines [complexity: high]
3. src/util/Formatter.cc — 45%, 88 uncovered lines [complexity: medium]
4. src/algo/Helper.hh — 0%, header-only [complexity: low]
5. src/algo/Config.h — 0%, data class [complexity: low]

Run /ut-pilot:continue to process this batch.
Run /ut-pilot:path <dir> to focus on a specific directory first.
```

### If All Done

```
All <N> source files are at >90% coverage (or recorded as [BuildFail] in UT_RULES.md ## Project Gotchas).
- At >90%: X files
- [BuildFail] (see UT_RULES.md ## Project Gotchas): W files

Run /ut-pilot:status again after any source changes to check for regressions.
```
