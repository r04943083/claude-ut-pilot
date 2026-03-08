---
description: "Automatically loop continue until all files reach >90% coverage"
---

# /ut-pilot:auto — Autonomous Coverage Loop

**Responsibility**: Run continue batches in a loop WITHOUT pausing for user input between batches. Stop only when a terminal condition is reached.

## Initialization

Read `UT_RULES.md`. If not found, stop: "UT_RULES.md not found. Run /ut-pilot:init first."

Parse: `parallel_agents`, `files_per_batch`, `strategy`, `current_focus`.
Read `## Max Coverage Files` section → build `maxcov_files` set (all file paths listed there).

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

Collect all `- [ ]` entries. Exclude: files in `maxcov_files` set, files marked `[BuildFail]`.

Apply `current_focus` filter if set.

**Terminal check — exit if**:
- Remaining (not excluded) == 0 → Exit: All Done

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
3. Read UT_RULES.md for gotchas and conventions
4. Write tests targeting >90% line coverage per file
5. Update CMakeLists.txt
6. If you cannot reach >90% due to system dependencies, write what IS testable,
   then add an entry to UT_RULES.md '## Max Coverage Files' with reason:
   `- FileName.cc (MaxCov: X%) — reason`
7. If you discover a gotcha, append it to UT_RULES.md '## Project Gotchas'.
Report: files created/modified and expected coverage level.
```

Wait for all agents to complete.

**D. Verify**

```bash
cd <test_root> && bash run_tests.sh
```

If build fails: fix (up to 3 attempts). If still failing after 3 attempts, mark affected files `[BuildFail]` in TODO.md.

```bash
cd <test_root> && bash coverage.sh
```

This regenerates TODO.md. Count how many files newly crossed 90%.

**E. Update maxcov_files**

Re-read `UT_RULES.md` `## Max Coverage Files` section. Any newly added entries = progress for this batch.

**F. Progress Report**

```
--- Batch <N> complete ---
Processed: <list of files>
Newly at >90%: X files
Newly documented MaxCov: Y files
Overall: <covered + maxcov> / <total> files resolved
Remaining: <Z> files need tests
--------------------------
```

Increment `batch_number`, add to `total_files_processed`.

**G. Persist Gotchas**

Ensure any new gotchas discovered this batch are saved to UT_RULES.md.

**H. No pause — go to next iteration immediately.**

---

## Stop Conditions

| Condition | Exit Message |
|-----------|-------------|
| All files ≥90% or in MaxCov set or [BuildFail] | "All Done" report |

## Exit Report

### All Done

```
## Auto Complete

Processed <N> batches, <M> total files.
All files are now either at >90% coverage or documented in Max Coverage Files.

Final coverage:
- At >90%: X files
- MaxCov documented: Y files
- [BuildFail]: W files

Run /ut-pilot:status for a full breakdown.
```
