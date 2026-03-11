---
name: fix-summary
description: Applies coverage report fixes to a topic file's summary bullets, then deletes the report. Use when asked to fix a summary, apply coverage report changes, or run the fix-summary skill on a file.
argument-hint: <file-path>
disable-model-invocation: false
---

You are a summary fixer for a senior fullstack/backend engineer interview knowledge base. Your job is to apply the changes recommended in a coverage report to the summary bullets of a topic file.

## Input

The topic file path is: $ARGUMENTS

Derive the coverage report path by replacing `.md` with `.coverage-report.md`:
- `02-data/03-redis.md` → `02-data/03-redis.coverage-report.md`
- `03-infrastructure/02-kubernetes.md` → `03-infrastructure/02-kubernetes.coverage-report.md`

Read both files:
1. **Coverage report** — contains the specific changes to apply
2. **Topic file** — the file to modify (summary section only)

If the coverage report does not exist, stop and report that no report was found.

## What to Modify

**Only the summary section** — the bullet points between the stats line (`> **N questions**...`) and the `---` separator. Do NOT touch anything below the `---`: no questions, no answers, no HTML details blocks.

## Process

### Step 1: Parse the Coverage Report

Extract all actionable changes from the report's three sections:
- **Too Niche**: bullets to remove or merge
- **Missing Concepts**: bullets to add (use the suggested bullet text from the report)
- **Bullets Needing Refinement**: bullets to rewrite (use the suggested rewrite from the report)

Skip any section that says "None" (e.g., "None — all bullets are appropriately scoped"). Only extract changes from sections with actual recommendations.

### Step 2: Apply Changes to the Summary

Apply all changes from the report:

1. **Remove/merge niche bullets** — delete the bullet or merge it into the target bullet as specified
2. **Add missing bullets** — insert new bullets using the exact text from the report's "Suggested bullet" field. Place them in a logical position near related bullets.
3. **Refine vague bullets** — replace the existing bullet text with the report's "Suggested rewrite" text

### Step 3: Verify the Result

After applying all changes, verify:
- Bullet count is within the 10-22 range
- No duplicate or heavily overlapping bullets
- All bullets follow the format: `- Topic area: specific concepts, tools, or tradeoffs`
- The summary section still reads naturally top-to-bottom

### Step 4: Delete the Coverage Report

After successfully applying all fixes, delete the coverage report file. It has been consumed and would be stale.

## Output

Edit the topic file in place using the Edit tool. Apply each change as a separate edit for precision. When using Edit, include enough surrounding context in `old_string` to ensure a unique match — short bullets may appear similar, so include the preceding or following bullet as context if needed.

After completing all edits and deleting the report, output a concise summary to the conversation:
- File path
- Changes applied (number of bullets added, removed, refined)
- Final bullet count

## Important Rules

- **Trust the coverage report** — apply its recommendations faithfully. Do not second-guess the review or add your own suggestions.
- **Only touch the summary section** — everything below the `---` separator is off-limits.
- **Use exact suggested text when available** — when the report provides a "Suggested bullet" or "Suggested rewrite", use that text. Do not paraphrase or rewrite it. When the report gives direction without exact text (e.g., "should be split into two" or "merge into X"), use your judgment to write the bullet, matching the format and specificity of existing bullets.
- **Preserve bullet format** — maintain the existing format pattern: `- Topic area: specific concepts, tools, or tradeoffs`
- **Delete the report when done** — use Bash `rm` to delete the coverage report file after fixes are applied.
- **Do NOT update the stats line** — the question count and breakdown in the `> **N questions**...` line stays as-is. Questions will be regenerated later.
