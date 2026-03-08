---
description: "Automatically loop continue until all files reach >90% coverage"
---

# /ut-pilot:auto — Autonomous Coverage Loop

**Responsibility**: Run continue batches in a loop WITHOUT pausing for user input between batches. Stop only when a terminal condition is reached.

## Initialization

Read `UT_RULES.md`. If not found, stop: "UT_RULES.md not found. Run /ut-pilot:init first."

Parse: `parallel_agents`, `files_per_batch`, `strategy`, `current_focus`.

Initialize loop state:
- `batch_number` = 0
- `total_files_processed` = 0

## The Loop

Repeat the following until a stop condition is met. **Do NOT ask for user confirmation between iterations. Do NOT pause. Proceed immediately to the next batch.**

---

### Each Iteration:

**A. Load TODO.md**

Read `TODO.md`. Format: directory sections with bullet entries:
- `- [x] Foo.cc (94% - covered)` → skip (≥90%)
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` → actionable
- `- [ ] Baz.cc (0%, ? uncov - no tests)` → actionable

Collect all `- [ ]` entries. Exclude files recorded as `[BuildFail]` in `UT_RULES.md ## Project Gotchas`. Do NOT exclude any other files — every file must be attempted.

Apply `current_focus` filter if set.

**Terminal check — exit if**:
- No `- [ ]` entries remain (all files are `- [x]` at ≥90%, or noted as BuildFail in UT_RULES.md) → Exit: All Done

**B. Score and Select Batch**

Apply **complex-first** scoring to rank remaining files (higher = process first):
- Has `.cc`: +2
- `.cc` >100 lines: +1
- Uncovered lines above median: +1
- Many internal includes: +1 per 3 includes (max +2)

Select top `files_per_batch` files. **Do NOT auto-mark any file as [Skip].**

**C. Assign and Spawn Agents**

Divide files evenly across `parallel_agents` agents. Spawn agents in parallel using the Agent tool.

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
3. Read UT_RULES.md for gotchas and conventions (especially ## Project Gotchas for existing fixture patterns)
4. Write tests targeting >90% line coverage per file
   - For files with system-context dependencies: search other _ut.cc files in the same module directory for existing fixtures/wrappers and reuse them
   - The project is guaranteed to compile — there is always a way to write at least one passing test
5. Update CMakeLists.txt
6. Do NOT add entries to UT_RULES.md '## Max Coverage Files'. Every file can be tested.
7. If you discover a gotcha or a useful fixture pattern, append it to UT_RULES.md '## Project Gotchas'.
Report: files created/modified and expected coverage level.
```

Wait for all agents to complete.

**D. Verify**

```bash
cd <test_root> && bash run_tests.sh
```

If build fails: fix (up to 3 attempts). If still failing after 3 attempts, record affected files as `[BuildFail] FileName.cc — <error>` in `UT_RULES.md ## Project Gotchas` (do NOT write to TODO.md — it gets overwritten by coverage.sh).

```bash
cd <test_root> && bash coverage.sh
```

This regenerates TODO.md. Count how many files newly crossed 90%.

**E. Progress Report**

```
--- Batch <N> complete ---
Processed: <list of files>
Newly at >90%: X files
Overall: <covered> / <total> files at >90%
Remaining: <Z> files need tests
--------------------------
```

Increment `batch_number`, add to `total_files_processed`.

**F. Persist Gotchas and Build Failures**

- Save any new gotchas discovered this batch to `UT_RULES.md ## Project Gotchas`.
- If any file had an unrecoverable build failure, record it in `UT_RULES.md ## Project Gotchas` as:
  `- [BuildFail] FileName.cc — <error summary>` (do NOT mark TODO.md, it gets overwritten by coverage.sh)

**G. No pause — go to next iteration immediately.**

---

## Stop Conditions

| Condition | Exit Message |
|-----------|-------------|
| All `- [ ]` entries gone (files reached ≥90% or are recorded as `[BuildFail]` in UT_RULES.md) | "All Done" report |

## Exit Report

### All Done

```
## Auto Complete

Processed <N> batches, <M> total files.
All files are now at >90% coverage (or recorded as [BuildFail] in UT_RULES.md).

Final coverage:
- At >90%: X files
- [BuildFail] (see UT_RULES.md ## Project Gotchas): W files

Run /ut-pilot:status for a full breakdown.
```
