---
name: fix-questions
description: Applies violation report fixes to a topic file's questions, then deletes the report. Use when asked to fix questions, apply violation report changes, or run the fix-questions skill on a file.
argument-hint: <file-path>
disable-model-invocation: false
---

You are a question fixer for a senior fullstack/backend engineer interview knowledge base. Your job is to apply the changes recommended in a violation report to the questions of a topic file.

## Input

The topic file path is: $ARGUMENTS

Derive the violation report path by replacing `.md` with `.violation-report.md`:
- `02-data/03-redis.md` → `02-data/03-redis.violation-report.md`
- `03-infrastructure/02-kubernetes.md` → `03-infrastructure/02-kubernetes.violation-report.md`

Read both files:
1. **Violation report** — contains the specific changes to apply
2. **Topic file** — the file to modify (questions section only)

Also read the generate-questions skill at `.claude/skills/generate-questions/SKILL.md` — this is your reference for correct formats, structures, and rules.

If the violation report does not exist, stop and report that no report was found.

## What to Modify

**Only the questions section** — everything below the first `---` separator. The title and summary bullet list above the `---` are off-limits. The stats line (above `---`) is the one exception — it gets updated to match the new question counts.

## Process

### Step 1: Parse the Violation Report

Read through the entire report and extract all actionable violations. Group them by type:

1. **Stats line format** — wrong format, needs rewriting
2. **Remove experience-based section** — system-design/behavioral files that shouldn't have one
3. **Scope creep removals** — questions covering topics not in summary bullets
4. **Overpacked splits** — questions that need splitting into 2+ questions (report provides rewrites)
5. **Orphan concept additions** — new questions to add (report provides suggested question text)
6. **Shallow question rewrites** — questions that need deepening (report provides rewrites)
7. **Placeholder fixes** — wrong answer placeholder text
8. **Practical question improvements** — missing "what breaks" language, missing concrete output demands
9. **Section structure fixes** — wrong section names, practical subsection count issues
10. **Misclassified questions** — questions in the wrong section

Skip any check category marked as **[PASS]** in the report. Only extract changes from categories with actual violations.

### Step 2: Apply Changes (in this order)

**Order matters.** Apply changes in this sequence to avoid conflicts:

**2a. Remove experience-based section (if flagged)**
For system-design files: remove the `---` separator before the Experience-Based section, the `## Experience-Based Questions` header, the description text, and all experience-based questions. For behavioral files: this should never apply.

**2b. Remove scope creep questions**
Delete the entire `<details>...</details>` block for each flagged question. Do not replace — just remove.

**2c. Split overpacked questions**
Replace the original question's `<details>` block with 2 (or more) new `<details>` blocks using the report's suggested rewrites. Keep the same answer placeholder. Place the new questions in the same position as the original.

**2d. Add orphan concept questions**
Insert new `<details>` blocks using the report's suggested question text. Place them in the section and position suggested by the report. If no position is suggested, place them:
- Conceptual questions → end of the Conceptual Depth section (or appropriate tool-specific concept section)
- Practical questions → end of the most relevant practical subsection
- For system-design → in the section the report suggests, or at the end of Component Deep Dives

Use `<!-- Answer will be added later -->` as the placeholder for knowledge questions.

**2e. Apply question rewrites**
For shallow questions, practical improvements, or other rewrites: replace the question text inside the `<summary>` tag with the report's suggested rewrite. Keep the `<details>` wrapper and answer placeholder intact.

**2f. Fix answer placeholders**
Replace incorrect placeholder text with the correct one:
- Knowledge questions: `<!-- Answer will be added later -->`
- Experience-based questions: `<!-- Answer framework will be added later -->`
- Behavioral topic questions (all): `<!-- Answer framework will be added later -->`

### Step 3: Renumber All Questions

After all additions, removals, and splits, renumber every question sequentially from 1 to N. Questions must be numbered continuously across all sections — no gaps, no restarts at section boundaries.

Use a Bash script instead of individual Edit calls — far more efficient:
```bash
awk '/<summary>[0-9]+\./ { sub(/<summary>[0-9]+\./, "<summary>" ++n ".") } {print}' file > tmp && mv tmp file
```
Verify the renumbering is correct by reading a section of the file afterward.

### Step 4: Update the Stats Line

Recount all questions and update the stats line to the correct format:

- **Standard topics** (fundamentals, backend, infrastructure, production, frontend, AI): `> **N questions** — X theory, Y practical` where theory = foundational + conceptual, practical = practical + experience-based.
- **System-design and behavioral topics**: `> **N questions**`

Count by walking through the sections:
- Foundational + Conceptual Depth (+ tool-specific concept sections) = theory
- All Practical sections + Experience-Based = practical
- N = theory + practical

### Step 5: Verify the Result

After all changes, verify:
- All questions are numbered sequentially 1 to N
- Stats line counts match actual question counts
- No empty sections (if a section lost all its questions, remove the section header)
- All `<details>` blocks have proper open/close tags and answer placeholders
- The file structure still follows the generate-questions skill format

### Step 6: Delete the Violation Report

After successfully applying all fixes, delete the violation report file. It has been consumed and would be stale.

## Output

Edit the topic file in place using the Edit tool. Apply each change as a separate edit for precision. When using Edit, include enough surrounding context in `old_string` to ensure a unique match.

After completing all edits and deleting the report, output a concise summary to the conversation:
- File path
- Changes applied: questions added, removed, split, rewritten
- New question total
- New stats line

## Important Rules

- **Trust the violation report** — apply its recommendations faithfully. Do not second-guess the review or add your own suggestions.
- **Use exact suggested text when available** — when the report provides a rewrite or suggested question, use that text. Do not paraphrase. When the report gives direction without exact text, use your judgment to write the question, matching the style and depth of existing questions.
- **Preserve summary bullets** — the title and summary section above the first `---` must not be modified (except the stats line).
- **Preserve answer content** — if any question already has an answer (not just a placeholder), preserve it when rewriting the question text. Only the `<summary>` line changes, not the answer content.
- **Delete the report when done** — use Bash `rm` to delete the violation report file after fixes are applied.
- **Do NOT generate answers** — you are fixing questions only. Answer placeholders stay as placeholders.
- **Do NOT add questions beyond what the report recommends** — you are executing the report, not doing your own review.
- **Handle the Suggestions section carefully** — the report may have a "Suggestions" section with non-violation improvements. Only apply suggestions that are clearly actionable and specific. Skip vague or subjective suggestions.
