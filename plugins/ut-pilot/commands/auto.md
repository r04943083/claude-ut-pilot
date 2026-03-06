---
description: "Automatically loop continue until all files reach >90% coverage"
---

Use the ut-pilot skill to execute Auto mode.

Repeat the Continue workflow in a loop:
1. Run the Continue mode (pick next 3-5 uncovered files, write tests, build, verify)
2. After each batch, report progress
3. Check if there are still uncovered files with <90% coverage
4. If yes, start the next batch immediately without waiting for user input
5. Stop only when:
   - All files have >90% coverage, OR
   - All remaining files are classified as "Skip" (deep dependencies), OR
   - A build/test failure cannot be resolved after 3 attempts

Do NOT ask for confirmation between batches. Just keep going.
