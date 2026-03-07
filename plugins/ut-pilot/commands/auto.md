---
description: "Automatically loop continue until all files reach >90% coverage"
---

# /ut-pilot:auto — Autonomous Coverage Loop

**Responsibility**: Run continue batches in a loop WITHOUT pausing for user input between batches. Stop only when a terminal condition is reached.

## Initialization

Read `UT_RULES.md`. If not found, stop: "UT_RULES.md not found. Run /ut-pilot:init first."

Parse: `parallel_agents`, `files_per_batch`, `strategy`, `current_focus`.

Initialize loop state:
- `consecutive_no_progress` = 0
- `total_files_processed` = 0
- `batch_number` = 0

## The Loop

Repeat the following until a stop condition is met. **Do NOT ask for user confirmation between iterations. Do NOT pause. Proceed immediately to the next batch.**

---

### Each Iteration:

**A. Load TODO.md**

Read `TODO.md`. Identify all files with status "Needs Tests" and not marked `[Skip]` or `[BuildFail]`.

Apply `current_focus` filter if set.

**Terminal check — exit if**:
- Zero files remain (all covered or all skipped) → Exit: All Done
- All remaining are `[Skip]` or `[BuildFail]` → Exit: Nothing More to Do

**B. Score and Select Batch**

Same scoring logic as continue.md:

**complex-first**: Score each file. Higher = process first.
- Has `.cc`: +2
- `.cc` >100 lines: +1
- Uncovered lines above median: +1
- Many internal includes: +1 per 3 (max +2)
- Deep system dependency (PlacerDB-level): mark `[Skip]`, exclude

Select top `files_per_batch` files.

**C. Assign and Spawn Agents**

Divide files across `parallel_agents` agents (1-2 files each).

Spawn agents in parallel using the same agent prompt as continue.md:
- Source file content (.h + .cc)
- Relevant CMakeLists.txt patterns
- UT_RULES.md Configuration + Gotchas
- Write Test Workflow from ut-pilot SKILL.md

Wait for all agents to complete.

**D. Verify**

```bash
cd <test_root> && bash run_tests.sh
```

If build fails: fix (up to 3 attempts). If still failing, mark affected files `[BuildFail]` in TODO.md.

```bash
cd <test_root> && bash coverage.sh
```

This regenerates TODO.md. Count how many files newly crossed 90%.

**E. Batch Progress Report** (print after each batch, then continue immediately)

```
--- Batch <N> complete ---
Processed: <list of files>
Newly at >90%: X files
Still below 90%: Y files in this batch
Overall: <total covered> / <total> files at >90%
Remaining: <Z> files need tests
--------------------------
```

**F. Check Progress**

If newly-at-90% count for this batch = 0:
- Increment `consecutive_no_progress`
- If `consecutive_no_progress` >= 3: exit → Stuck

If any progress: reset `consecutive_no_progress` = 0

Increment `batch_number`, add to `total_files_processed`.

**G. Persist Gotchas**

Append any new gotchas discovered this batch to UT_RULES.md.

**H. No pause — go to next iteration immediately.**

---

## Stop Conditions

| Condition | Exit Message |
|-----------|-------------|
| All files ≥90% coverage | "All done" report (see below) |
| All remaining files are [Skip] or [BuildFail] | "Nothing more to process automatically" report |
| 3 consecutive batches with 0 new files reaching 90% | "Stuck" report |

## Exit Reports

### All Done
```
## Auto Complete — All Files Covered

Processed <N> batches, <M> total files.
All files now at >90% coverage (or classified Skip/BuildFail).

Final coverage: <X>/<Y> source files at >90%
Skipped (deep deps): <Z> files
Build failures: <W> files

Run /ut-pilot:status for a full breakdown.
```

### Nothing More to Process
```
## Auto Stopped — No Actionable Files Remain

<X>/<Y> files at >90%
<Z> files marked [Skip] (deep dependencies, require full system context)
<W> files marked [BuildFail] (build infrastructure not yet ready)

To handle skipped files: set up mocking infrastructure, then run /ut-pilot:continue manually.
```

### Stuck (no progress for 3 batches)
```
## Auto Stopped — No Progress for 3 Consecutive Batches

Last files attempted:
- <list>

Possible causes:
- Build errors that need manual investigation
- Test files exist but coverage tool not measuring them
- Files require mocking infrastructure not yet in place

Check the build output above. Fix the blocking issue, then resume with /ut-pilot:continue.
```
