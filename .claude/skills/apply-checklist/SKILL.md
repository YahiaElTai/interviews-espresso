---
name: apply-checklist
description: Cross-references a topic file against checklists.md and applies missing coverage — adds summary bullets and questions for uncovered checklist items. Use when asked to apply checklist, cross-reference checklist, or run the apply-checklist skill on a file.
argument-hint: <file-path>
disable-model-invocation: false
---

You are a checklist applier for a senior fullstack/backend engineer interview knowledge base. Your job is to cross-reference a topic file against the checklist entries in `checklists.md`, identify gaps, and apply changes directly — adding missing summary bullets and questions in one pass.

## Input

The topic file path is: $ARGUMENTS

Read these files:
1. **`checklists.md`** — the cross-reference checklists (at the repo root)
2. **The topic file** — the file to check and modify
3. **`.claude/skills/generate-questions/SKILL.md`** — your reference for question format, style, and structure rules

## Process

### Step 1: Extract Relevant Checklist Items

Scan `checklists.md` for all sections that reference the target file. Checklist sections use "Check against:" lines that list file paths. Collect every checklist item that maps to this file.

If the target file is not referenced in any checklist section, stop and report: "No checklist entries reference this file."

### Step 2: Audit Current Coverage

For each checklist item, determine whether it is already covered:

1. **Check the summary bullets** — is the concept mentioned (even partially) in an existing bullet?
2. **Check the questions** — is there a question that covers this concept?

Mark each checklist item as one of:
- **Covered** — both summary and questions address it adequately
- **Summary gap** — not in the summary bullets (may or may not have a question)
- **Question gap** — in the summary but no question covers it
- **Missing entirely** — neither summary nor questions address it

Skip items marked as "Covered."

### Step 3: Plan Changes

For each gap, decide the minimal change needed:

**Summary gaps and missing items:**
- Write a new bullet OR extend an existing bullet to include the concept
- Prefer extending an existing bullet if the concept is closely related to one already there
- New bullets must follow the format: `- Topic area: specific concepts, tools, or tradeoffs`

**Question gaps:**
- Write a new question OR modify an existing question to cover the concept
- Prefer modifying an existing question if the concept fits naturally (e.g., adding a sub-point to a related question)
- Only add a new question if the concept is distinct enough to warrant its own question
- New questions must follow the generate-questions skill rules: WHY > HOW > WHAT, no shallow questions, proper depth

### Step 4: Apply Summary Changes

Edit the summary section (bullets between the stats line and the `---` separator):
- Add new bullets in a logical position near related existing bullets
- Extend existing bullets by appending the new concepts naturally
- Do NOT remove or rewrite existing bullets unless extending them

### Step 5: Apply Question Changes

Edit the questions section (everything below the `---` separator):

**Adding new questions:**
- Place the new question in the appropriate section (Conceptual Depth for WHY questions, the relevant Practical section for HOW questions)
- Use the `<details>/<summary>` format with `<!-- Answer will be added later -->` placeholder
- Insert at a logical position within the section (near related questions)

**Modifying existing questions:**
- Extend the question text in the `<summary>` tag to include the new concept
- Only do this when the concept fits naturally — don't overpack questions (the generate-questions skill says: if a question has 3+ distinct sub-topics joined by "and," it's too big)
- If a question already has an answer (not just a placeholder), preserve the answer content. Add a note `<!-- TODO: Update answer to also cover [new concept] -->` after the existing answer content

### Step 6: Renumber Questions

After all additions, renumber every question sequentially from 1 to N. Questions must be numbered continuously across all sections — no gaps, no restarts at section boundaries.

Use a Bash script instead of individual Edit calls — far more efficient:
```bash
awk '/<summary>[0-9]+\./ { sub(/<summary>[0-9]+\./, "<summary>" ++n ".") } {print}' file > tmp && mv tmp file
```
Verify the renumbering is correct by reading a section of the file afterward.

### Step 7: Update Stats Line

Recount all questions and update the stats line:
- **Standard topics**: `> **N questions** — X theory, Y practical` where theory = foundational + conceptual, practical = practical + experience-based
- **System-design and behavioral topics**: `> **N questions**`

### Step 8: Verify

After all changes, verify:
- All questions are numbered sequentially 1 to N
- Stats line counts match actual question counts
- All `<details>` blocks have proper open/close tags and answer placeholders
- New bullets follow the existing format pattern
- No question was overpacked (3+ distinct sub-topics)

## Output

Edit the topic file in place using the Edit tool. Apply each change as a separate edit for precision. When using Edit, include enough surrounding context in `old_string` to ensure a unique match.

After completing all edits, output a structured summary:

```
## Apply Checklist: [file path]

### Checklist sections checked
- [list each checklist section name that references this file]

### Coverage before changes
- Covered: N items
- Gaps found: N items

### Changes applied
- Summary bullets added: N
- Summary bullets extended: N
- Questions added: N
- Questions modified: N
- New question total: N
- New stats line: [exact stats line]

### Details
- [For each change: what checklist item it addresses, what was added/modified]
```

## Important Rules

- **Minimal changes** — only add what the checklist says is missing. Do not restructure, rewrite, or "improve" existing content.
- **Preserve existing answers** — if a question has an answer filled in, do not lose it. When modifying such a question, keep the answer and add a TODO comment for updating it.
- **Respect file scope** — if a checklist item says "check against file X" but you're running on file Y, skip it even if the concept seems relevant. Each file has its own scope.
- **No duplicate coverage** — before adding a question, verify that no existing question already covers the concept (even partially or from a different angle). If it does, mark it as "Covered" and skip.
- **Follow question quality rules** — new questions must meet the same bar as generate-questions output: WHY > HOW > WHAT, no shallow questions, demand concrete output in practical questions.
- **Don't add niche items** — the checklist may list many items. Only add ones that pass the relevance test: would understanding this concept make a senior backend engineer meaningfully better at their job — does it change how they design, debug, or operate systems? If it's trivia they'd look up once and forget, or a detail that libraries/frameworks handle invisibly, skip it and note why.
- **Respect file scope strictly** — if a checklist item is a generic concept (not specific to the file's technology) and another file in the repo is a better home for it, skip it even if the checklist points here. For example, generic web security topics (path traversal, CSRF, timing attacks) belong in the auth/security file, not in a language-specific file like Node.js. Each file should only gain concepts that are specific to its domain.
- **Extend over add** — prefer extending an existing bullet or question over creating a new one. This keeps file size manageable and avoids fragmentation.
