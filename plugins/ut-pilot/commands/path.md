---
description: "Set the focused directory for continue — persists to UT_RULES.md"
---

# /ut-pilot:path — Set or Clear Current Focus

**Arguments**: `$ARGUMENTS` (target path, or empty to clear focus)

## Step 1: Load UT_RULES.md

Read `UT_RULES.md` from the test_root (search: project root, `tests/ut/`, `test/ut/`, `tests/`, `test/`).

If not found: stop and say "UT_RULES.md not found. Run /ut-pilot:init first."

Parse `source_root`, `test_root`, `strategy`, and current `current_focus`.

## Step 2: Resolve the Target Path

**If `$ARGUMENTS` is empty**: clear focus mode (go to Step 2b).

**If `$ARGUMENTS` is provided**:

Try to resolve it in this order:
1. As an absolute path — check if it exists
2. Relative to `source_root` — check if `<source_root>/$ARGUMENTS` exists
3. Relative to the current working directory — check if it exists

If the path does not exist after all attempts: stop and say "Path not found: $ARGUMENTS. Please provide a path relative to source_root (<source_root>) or an absolute path."

Set `resolved_path` to the absolute path found.

**Step 2b — Clear focus**:

Set `resolved_path` to `""` (empty string). This returns continue to global mode.

## Step 3: Update UT_RULES.md

Edit the `current_focus` line in `UT_RULES.md`:

```
- current_focus: <resolved_path>
```

If `$ARGUMENTS` was empty:
```
- current_focus: ""
```

Use exact line replacement — find the line starting with `- current_focus:` and replace it.

## Step 4: Refresh TODO.md and Scan the Target Directory

**If focus was cleared**: skip to Step 5 (clear mode report).

First, ensure TODO.md is up to date:
- If `<test_root>/build_cov/coverage_filtered.info` exists, run:
  ```bash
  cd <test_root> && bash gen_todo.sh
  ```
  to regenerate TODO.md from the latest coverage data.
- If coverage data does not exist yet, note this in the report (suggest running `/ut-pilot:init` first).

Then read TODO.md and classify all files under `resolved_path`:
- `- [x]` entries → Covered (≥90%)
- `- [ ]` entries → Needs Tests
- Source files not in TODO.md at all → also count as Needs Tests (not yet instrumented)

Count:
- Total source files in directory
- Already covered (≥90%)
- Needs tests (all `- [ ]` entries + any uninstrumented files)

Apply the configured `strategy` scoring to order the "Needs Tests" files — show the top 10.

## Step 5: Report

**If focus was set**:
```
Focus set: <resolved_path>

Directory status:
- Total source files: N
- Already at >90% coverage: X
- Need tests: Y

Next targets (complex-first):
1. path/to/File1.cc — 0%, ~180 lines
2. path/to/File2.cc — 34%, ~95 uncovered lines
3. path/to/File3.cc — 0%, ~72 lines
...

UT_RULES.md updated: current_focus = <resolved_path>

Next /ut-pilot:continue will only process files from this directory.
Run /ut-pilot:path (no args) to return to global mode.
```

**If focus was cleared**:
```
Focus cleared. UT_RULES.md updated: current_focus = ""

Next /ut-pilot:continue will process files from the entire project (global mode).
```
