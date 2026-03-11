---
name: distill
description: Trims a topic file to its essential 60-70% scope by surgically deleting niche summary bullets and their corresponding questions. Used to create the interview-espresso version of a file.
argument-hint: <file-path>
disable-model-invocation: false
---

You are distilling a comprehensive interview prep file down to its essential core. Your job is to apply the 80/20 principle: keep the essential 60-70% of summary bullets that cover the most important, most commonly asked, and most foundational knowledge — and cut the rest. Then drop any questions whose topics were removed.

**You only DELETE via surgical edits. You never add, rewrite, or modify content.** Your output is a strict subset of the input.

## Input

The file to distill: `$ARGUMENTS`

This file lives in the current repo. Read it from the current working directory.

## The Distillation Principle

The main interviews repo is the comprehensive gold standard — 20-40 questions per topic covering everything a senior engineer should know. The espresso repo is for "I have an interview next week" — it should cover the essential must-know concepts with the same answer quality but narrower scope.

**Same depth, narrower scope.** You're not making answers shorter or simpler. You're deciding which topics to cover.

## Process

### Step 1: Read and Understand the File

Read the full file. Identify:
- The summary bullet points (between the title and the `---` separator)
- The stats line (e.g., `> **37 questions** — 18 theory, 19 practical`)
- All knowledge questions (in `<details>` blocks)
- The experience-based questions section (if present)

### Step 2: Classify Each Summary Bullet

For each summary bullet, classify it as **KEEP** or **CUT** using these criteria:

**KEEP if ANY of these are true:**
- It's a foundational concept that other topics build on (e.g., "event loop phases" for Node.js)
- It's commonly asked in interviews for this topic (the "greatest hits")
- It's essential for day-to-day production work
- Removing it would leave a visible gap — someone studying this file would notice it's missing
- It's a prerequisite for understanding other kept bullets

**CUT if ALL of these are true:**
- It's niche, specialist-level, or rarely asked in interviews
- It's an advanced optimization or edge case that most engineers don't encounter
- It's "nice to know" but not "must know"
- The file still makes sense without it — no gap is created

**Aim for cutting ~30-40% of bullets.** If a topic is already lean (10-12 bullets), you might only cut 3-4. If it's dense (18-22 bullets), you might cut 6-8. These percentages are guidelines, not goals — don't force a number if it means cutting essential content or keeping filler. Let the topic dictate the right cut depth.

### Step 3: Plan the Cuts

Before making any changes, output your plan:

```
# Distill Plan: [File Name]

## Bullets to CUT (N of M total)
- [bullet text] — [why: niche / rarely asked / advanced optimization / nice-to-know]
...

## Questions to DROP
- Q[N]: "[question summary]" — maps to cut bullet: "[bullet text]"
...

## Experience Questions to DROP (if any)
- Q[N]: "[question summary]" — [why: too advanced / maps to cut content]
Or: None — all experience questions are essential

## Expected Result
- Summary bullets: M → N (cut X)
- Knowledge questions: M → N (cut X)
- Experience questions: M → N (cut X)
```

Only list what you're cutting — keeps are implicit (everything else). Don't waste tokens justifying keeps.

**Then immediately proceed to Step 4.** Do not stop or wait for approval.

### Step 4: Map Questions to Cut Bullets

For each cut bullet, identify ALL questions that primarily test knowledge from that bullet. A question should be dropped if:
- Its core topic was in a cut bullet
- It can't be answered without knowledge from a cut bullet
- It's primarily about a concept that was cut

A question should be KEPT even if it touches a cut bullet, as long as:
- Its primary topic is in a kept bullet
- It can be fully answered using only knowledge from kept bullets

### Step 5: Handle Experience-Based Questions

Review experience-based questions with the same principle:
- Keep all that relate to kept topics
- Drop only if the question is clearly about a cut topic or is too advanced/niche
- When in doubt, keep it — experience questions are broadly valuable

### Step 6: Apply Surgical Edits

Edit the file in place using surgical delete operations. Do NOT rewrite the entire file — that wastes tokens. Instead, make targeted edits:

1. **Delete cut summary bullets** — remove each cut bullet line from the summary section
2. **Re-read the file** — line numbers have shifted after bullet deletions. You MUST re-read before proceeding.
3. **Delete cut questions** — remove the entire `<details>...</details>` block for each dropped question (including the blank lines around it). Work from largest to smallest question number to minimize offset issues. Re-read periodically if making many deletions.
4. **Delete empty sections** — if all questions under a section header were dropped, remove that section header too
5. **Update the stats line** — edit the `> **N questions**...` line to reflect new counts
6. **Renumber remaining questions** — use a Bash script to renumber all `<summary>` tags sequentially. This is far more efficient than individual Edit calls. Example approach:
   ```bash
   # Renumber all questions sequentially starting from 1
   awk '/<summary>[0-9]+\./ { sub(/<summary>[0-9]+\./, "<summary>" ++n ".") } {print}' file > tmp && mv tmp file
   ```
   Verify the renumbering is correct by reading a section of the file afterward.

**Order of edits**: Delete bullets → re-read → delete question blocks → clean up empty sections → update stats → renumber via script → verify.

## Quality Checks

After all edits, verify:
- [ ] Every kept summary bullet has at least one corresponding question
- [ ] No kept question references a concept only covered by a cut bullet
- [ ] Question numbering is sequential with no gaps
- [ ] Stats line matches actual question counts
- [ ] The file reads coherently — no orphaned references to cut content
- [ ] Section headers match their contents (no empty sections)

## Important Rules

- **NEVER add content** — no new bullets, no new questions, no rewrites. Only delete.
- **NEVER modify question text** — keep exact wording. Only change the question number.
- **NEVER modify answer text** — if a question has an answer, keep it exactly as-is.
- **NEVER modify summary bullet text** — keep exact wording of kept bullets.
- **Preserve formatting exactly** — `<details>`, `<summary>`, markdown headers, separators.
- **The behavioral file (`10-behavioral/`)** — may not need much trimming. If all questions are essential, say so and make minimal cuts.
- **System design files (`07-system-design/`)** — trim the summary and questions, but preserve the design interview section structure.

## Report Back

After writing the distilled file, return:
- File path
- Summary bullets: before → after (what was cut)
- Knowledge questions: before → after (what was dropped)
- Experience questions: before → after (what was dropped, if any)
- Brief rationale for the most significant cuts
