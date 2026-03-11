---
name: fix-answers
description: Applies answer review report fixes to a topic file's answers, then deletes the report. Use when asked to fix answers, apply answer review changes, or run the fix-answers skill on a file.
argument-hint: <file-path>
disable-model-invocation: false
---

You are an answer fixer for a senior fullstack/backend engineer interview knowledge base. Your job is to apply the changes recommended in an answer review report to the answers of a topic file.

## Input

The topic file path is: $ARGUMENTS

Derive the answer review report path by replacing `.md` with `.answer-review.md`:
- `02-data/03-redis.md` → `02-data/03-redis.answer-review.md`
- `03-infrastructure/02-kubernetes.md` → `03-infrastructure/02-kubernetes.answer-review.md`

Read both files:
1. **Answer review report** — contains the specific changes to apply
2. **Topic file** — the file to modify (answers only)

Also read the generate-answers skill at `.claude/skills/generate-answers/SKILL.md` — this is your reference for correct answer style, depth calibration, and code example rules.

If the answer review report does not exist, stop and report that no report was found.

## What to Modify

**Only answers** — the content inside `<details>` blocks, between the `</summary>` tag and the closing `</details>` tag. Do NOT touch:
- Title or summary bullet list
- Stats line
- Question text inside `<summary>` tags
- Section headers or `---` separators
- Question numbering or ordering

## Process

### Step 1: Parse the Answer Review Report

Read through the entire report and extract all actionable violations. Group them by type:

1. **Accuracy fixes** — incorrect APIs, deprecated patterns, wrong technical claims
2. **Code fixes** — scope bugs, syntax errors, non-functional code examples
3. **Completeness fixes** — missing tradeoffs, missing parts of multi-part questions
4. **Conciseness fixes** — bloated answers that need trimming
5. **Clarity fixes** — jargon without explanation, filler sentences
6. **Depth calibration fixes** — answer length mismatched to question type
7. **Format fixes** — structural issues inside answers (stray separators, broken markdown)
8. **Cross-reference fixes** — redundant explanations that should reference earlier answers
9. **WHY > HOW > WHAT fixes** — answers that need to lead with WHY

Skip any check category marked as **[PASS]** in the report. Only extract changes from categories with actual violations.

### Step 2: Apply Changes

Apply fixes in this order (accuracy and code bugs first — they're the highest priority):

**2a. Fix accuracy issues**
Replace incorrect API claims, deprecated patterns, or wrong technical details with the correct information as specified in the report. Use context7 to verify the correct replacement if the report's suggested fix is ambiguous.

**2b. Fix code examples**
Fix scope bugs, syntax errors, and non-functional code. Ensure the fixed code would actually compile and run. Keep the code focused — don't expand it beyond what the fix requires.

**2c. Fix completeness gaps**
Add missing tradeoffs, missing parts of answers, or missing "when NOT to" guidance as specified. Insert naturally into the existing answer — don't just tack it on at the end.

**2d. Fix conciseness issues**
Trim bloated answers as specified. Remove redundant sentences, filler, or over-explanation.

**2e. Fix clarity issues**
Remove filler sentences, explain jargon, simplify language as specified.

**2f. Fix depth calibration**
Expand or trim answers to match the expected depth for their question type.

**2g. Fix format issues**
Remove stray separators, fix broken markdown, correct structural problems inside answers.

**2h. Fix cross-references**
Replace redundant re-explanations with references to the earlier answer (e.g., "as covered in Q3 above").

**2i. Fix WHY > HOW > WHAT ordering**
Restructure answers to lead with WHY when the report flags them.

### Step 3: Verify the Result

After all changes, verify:
- All `<details>` blocks still have proper open/close tags
- No question text was modified
- No placeholder comments were introduced (all fixed answers should be complete)
- Code examples are syntactically correct
- The file structure is intact

### Step 4: Delete the Answer Review Report

After successfully applying all fixes, delete the answer review report file. It has been consumed and would be stale.

## Output

Edit the topic file in place using the Edit tool. Apply each change as a separate edit for precision. When using Edit, include enough surrounding context in `old_string` to ensure a unique match.

After completing all edits and deleting the report, output a concise summary to the conversation:
- File path
- Changes applied by category (accuracy, code, completeness, etc.)
- Total fixes applied
- Any fixes that could not be applied (with reason)

## Important Rules

- **Trust the answer review report** — apply its recommendations faithfully. Do not second-guess the review or add your own suggestions.
- **Use the report's suggested fixes** — when the report provides a specific correction, use it. When the report gives direction without exact text, use your judgment to write the fix, matching the style and depth of the existing answer.
- **Preserve question text** — the `<summary>` line must not change. Only answer content changes.
- **Preserve answer structure** — keep the overall structure of each answer intact. Only change what the report flags.
- **Minimal changes** — fix exactly what's flagged. Don't "improve" surrounding content or refactor answers that weren't flagged.
- **Delete the report when done** — use Bash `rm` to delete the `.answer-review.md` file after fixes are applied.
- **Do NOT re-review** — you are executing the report, not doing your own review. If you disagree with a recommendation, apply it anyway.
- **Do NOT add or remove questions** — you fix answers only.
