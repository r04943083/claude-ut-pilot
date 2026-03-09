---
description: "Automatically loop continue until all files reach >90% coverage"
---

# /ut-pilot:auto — Autonomous Coverage Loop

**Responsibility**: Run continue batches in a loop WITHOUT pausing for user input between
batches. Stop only when a terminal condition is reached.

## Initialization

Read `UT_RULES.md`. If not found, stop: "UT_RULES.md not found. Run /ut-pilot:init first."

Parse: `parallel_agents`, `files_per_batch`, `strategy`, `current_focus`.

Initialize loop state:
- `batch_number` = 0
- `total_files_processed` = 0

## The Loop

Repeat the following until a stop condition is met. **Do NOT ask for user confirmation between
iterations. Do NOT pause. Proceed immediately to the next batch.**

---

### Each Iteration:

**A. Load TODO.md**

Read `TODO.md`. Format: directory sections with bullet entries:
- `- [x] Foo.cc (94% - covered)` → skip (≥90%)
- `- [ ] Bar.cc (0%, 617 uncov - needs tests)` → actionable
- `- [ ] Baz.cc (0%, ? uncov - no tests)` → actionable

If `TODO.md` does not exist or is stale:
- If `<test_root>/build_cov/coverage_filtered.info` exists:
  → run `cd <test_root> && bash gen_todo.sh <MODULE> <SOURCE_ROOT>` (fast, seconds)
- If `coverage_filtered.info` does not exist:
  → run `cd <test_root> && bash coverage.sh` (full rebuild, minutes)

Collect all `- [ ]` entries. Exclude files matching ANY of the following in `UT_RULES.md`:
- Recorded as `[BuildFail]` in `## Project Gotchas`
- Recorded as `[NoCode]` in `## Project Gotchas`
- Listed in `## Max Coverage Files` (already at documented coverage ceiling)

Do NOT exclude `[DeclOnly]` files — these are testable (write instantiation tests).

Do NOT exclude any other files — every remaining file must be attempted.

Apply `current_focus` filter if set.

**Terminal check — exit if**:
- No `- [ ]` entries remain (all files are `- [x]` at ≥90%, or excluded as above) → Exit: All Done

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

Wait for all agents to complete.

**D. Verify**

```bash
cd <test_root> && bash run_tests.sh
```

If build fails: fix (up to 3 attempts). If still failing after 3 attempts, record the affected
files in `UT_RULES.md ## Project Gotchas` (do NOT write to TODO.md — coverage.sh overwrites it):
```
- [BuildFail] FileName.cc — <error summary>
```

Only add `[BuildFail]` when the build actually fails (non-zero cmake/make exit code) after
3 fix attempts. A runtime crash is NOT a build failure. Low coverage is NOT a build failure.

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

Save any new gotchas discovered this batch to `UT_RULES.md ## Project Gotchas`.

When recording classifications:
- `[BuildFail] FileName.cc — <error>` — compilation/link failed after 3 fix attempts
- `[NoCode] FileName.cc — <reason>` — no executable code to instrument
- Files at coverage ceiling → add to `## Max Coverage Files` as `**[MaxCov] FileName.cc (X%, Y uncov)**: reason. Maximum achievable: X%.`

**G. No pause — go to next iteration immediately.**

---

## Stop Conditions

| Condition | Exit Message |
|-----------|-------------|
| All `- [ ]` entries gone (files reached ≥90%, or excluded as [BuildFail]/[NoCode] in UT_RULES.md ## Project Gotchas, or listed in ## Max Coverage Files) | "All Done" report |

## Exit Report

### All Done

```
## Auto Complete

Processed <N> batches, <M> total files.
All files are now at >90% coverage or documented as excluded in UT_RULES.md.

Final coverage:
- At >90%: X files
- [BuildFail] (see UT_RULES.md ## Project Gotchas): W files
- [NoCode] (see UT_RULES.md ## Project Gotchas): V files
- [MaxCov] (see UT_RULES.md ## Max Coverage Files): T files

Run /ut-pilot:status for a full breakdown.
```
