---
name: distill-light
description: Light distillation for core-stack topics — cuts only the truly tangential 10-15% while preserving senior-level depth. For primary tools (Node.js, TypeScript) where deep interview probing is expected.
argument-hint: <file-path>
disable-model-invocation: false
---

You are performing a **light distillation** on a core-stack interview prep file. Unlike standard distillation (which cuts 30-40%), you cut only the truly tangential 10-15% — tooling trivia, easily-Googled facts, and topics that belong in a different interview round entirely.

**You only DELETE via surgical edits. You never add, rewrite, or modify content.** Your output is a strict subset of the input.

## Input

The file to distill: `$ARGUMENTS`

This file lives in the current repo. Read it from the current working directory.

## The Light Distillation Principle

Core-stack topics are the candidate's primary tools. Interviewers **will** go deep here. Senior-level depth is exactly where a candidate differentiates — cutting it defeats the purpose.

**Keep ~85-90% of the content. Only cut what's genuinely tangential.**

## Process

### Step 1: Read and Understand the File

Read the full file. Identify:

- The summary bullet points (between the title and the `---` separator)
- The stats line (e.g., `> **37 questions** — 18 theory, 19 practical`)
- All knowledge questions (in `<details>` blocks)
- The experience-based questions section (if present)

### Step 2: Classify Each Summary Bullet

For each summary bullet, classify it as **KEEP** or **CUT**:

**CUT only if ALL of these are true:**

- Removing it creates zero gap — no other kept topic depends on it
- It's either tooling/config trivia that engineers look up (package managers, version managers, build tool comparisons) or belongs in a different interview round entirely (e.g., dedicated security round). Note: configuration that requires reasoning about tradeoffs and how settings interact (e.g., production tsconfig choices) is NOT trivia — it's a synthesis topic. Only cut config that's pure reference material.
- It tests breadth of awareness, not depth of understanding

**KEEP everything else.** The bar is: if a senior candidate saying "I don't know" to this topic would be a red flag, it stays. Senior-level depth is the whole point — don't cut it just because it's advanced.

**Aim for cutting only 2-4 bullets.** If you can't find 2 genuinely tangential bullets, that's fine — cut fewer. Never force cuts to hit a number.

### Step 3: Plan the Cuts

Before making any changes, output your plan:

```
# Distill-Light Plan: [File Name]

## Bullets to CUT (N of M total)
- [bullet text] — [why: tooling trivia / different round / easily Googled]
...

## Questions to DROP
- Q[N]: "[question summary]" — maps to cut bullet: "[bullet text]"
...

## Experience Questions to DROP (if any)
- Q[N]: "[question summary]" — [why]
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

Keep all experience questions unless one is exclusively about a cut topic. When in doubt, keep it.

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
   awk '/<summary>[0-9]+\./ { sub(/<summary>[0-9]+\./, "<summary>" ++n ".") } {print}' file > file.tmp && mv file.tmp file
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

## Report Back

After writing the distilled file, return:

- File path
- Summary bullets: before → after (what was cut)
- Knowledge questions: before → after (what was dropped)
- Experience questions: before → after (what was dropped, if any)
- Brief rationale for each cut
