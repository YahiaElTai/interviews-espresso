---
name: review-questions
description: Reviews a topic file's questions for quality, compliance with the generate-questions skill guidelines, and structural correctness. Outputs a violation report.
argument-hint: <file-path>
disable-model-invocation: false
---

You are a quality reviewer for a senior fullstack/backend engineer knowledge base. Your job is to review the questions in a topic file and produce a detailed violation report.

## Input

Read the file at path: $ARGUMENTS

Also read the question generation skill at: `.claude/skills/generate-questions/SKILL.md` — this contains all the rules the questions should comply with. Treat every rule as a requirement.

Also read the reference implementation at: `03-infrastructure/02-kubernetes.md` — this is the gold standard for question quality and structure.

## Review Process

You MUST check every rule below. For each check, either mark it as PASS or list specific violations with the question number(s) and a concrete fix suggestion.

### 1. Format & Structure Compliance

- [ ] Title and summary bullet list are preserved exactly (not modified from original)
- [ ] Stats line exists and format is correct: `> **N questions** — X theory, Y practical` for standard topics, or `> **N questions**` for system-design and behavioral topics. Theory = foundational + conceptual, practical = practical + experience-based.
- [ ] Stats line counts match actual question counts
- [ ] `---` separator after summary bullet list
- [ ] `---` separator before Experience-Based Questions section (N/A for system-design and behavioral — they have no separate experience section)
- [ ] Every question is in a `<details>/<summary>` block
- [ ] Every knowledge question has `<!-- Answer will be added later -->` placeholder
- [ ] Every experience question has `<!-- Answer framework will be added later -->` placeholder
- [ ] Questions are numbered sequentially from 1 to N across all sections (no restarts)
- [ ] No Table of Contents exists (TOC is explicitly forbidden)
- [ ] Section headers use `##` level

### 2. Topic Classification & Ratio

**Note:** `07-system-design/` and `10-behavioral/` topics use different structures (see Check 4). The foundational count, ratio, and experience-based count checks below apply to standard topics only — skip them for system design and behavioral files.

- [ ] Topic type is correctly classified (practical-heavy / balanced / conceptual-heavy) based on the classification list in the generate-questions skill
- [ ] Foundational questions: 2-3
- [ ] Total knowledge questions: 20-40
- [ ] Experience-based questions: 3-5
- [ ] Foundational + Conceptual counts toward conceptual side of ratio
- [ ] Ratio is within 5% of target for the topic type
- [ ] Report the actual ratio: `Actual: X% conceptual / Y% practical (target: A% / B%)`

### 3. Multi-Tool Detection

- [ ] If the title contains "&", "+", "and", or lists multiple tool names → it's a multi-tool topic
- [ ] If multi-tool: conceptual sections are split by tool (e.g., `## ToolA — Concepts`)
- [ ] If multi-tool: foundational introduces the overall stack, not individual tools
- [ ] If multi-tool: each tool subsection progresses naturally within itself
- [ ] If single-tool: uses standard `## Conceptual Depth` section

### 4. Section Structure

**Note:** `07-system-design/` topics use design interview sections (Requirements → Architecture → Deep Dives → Scaling). `10-behavioral/` topics use a flat list with no section headers. The progressive depth check below applies to standard topics only.

- [ ] Sections follow progressive depth: Foundational → Conceptual → Practical → Experience-Based (standard topics only)
- [ ] Practical sections have descriptive subtitles (not generic names)
- [ ] Practical sections are split into 2-4 subsections (or 1-2 for conceptual-heavy topics)
- [ ] Questions within each section build in complexity
- [ ] Category-specific structure is correct (check `07-system-design/` uses design interview flow, `10-behavioral/` uses flat list, etc.)

### 5. Question Quality — WHY > HOW > WHAT

For each question, verify:

- [ ] No "what is X?" standalone questions — every question demands the "why"
- [ ] Questions prioritize WHY → HOW → WHAT in that order
- [ ] Flag any question where the "what" isn't embedded naturally

**List every violating question number with a rewrite suggestion.**

### 6. No Shallow Questions

Test each question: **would a senior engineer need more than 30 seconds to give a complete answer?**

- [ ] Flag any question answerable in 1-2 sentences
- [ ] Flag any question that lacks depth, tradeoffs, or "why"
- [ ] Suggest how to deepen each flagged question (add "why", merge with another, add scenario)

**List every violating question number with explanation.**

### 7. No Overpacked Questions

Test each question: **could a senior engineer give a complete answer in under 5 minutes of speaking?**

- [ ] Flag any question with 3+ distinct sub-topics joined by "and"
- [ ] Flag any question trying to cover too much ground
- [ ] Suggest how to split each flagged question

**List every violating question number with explanation.**

### 8. Practical Question Rules

- [ ] Every practical question demands concrete output (code, config, YAML, commands, architecture)
- [ ] Questions use phrases like "show the YAML," "write a config," "walk through the exact commands"
- [ ] Practical questions explain what breaks if you skip something
- [ ] Scenario questions are specific (not "how do you debug X?" but "X is doing Y and you see Z")
- [ ] Code/config language matches the topic (TypeScript for backend, YAML for K8s, SQL for DB, etc.)

**List every violating question number with explanation.**

### 9. Real-World Relevance

- [ ] No hand-rolling what libraries solve (e.g., "implement a job queue from scratch" when Bull exists)
- [ ] No raw config for managed services (e.g., "write sentinel.conf" when teams use ElastiCache)
- [ ] No niche features (<10% of teams use) given dedicated questions
- [ ] Complexity ceiling is mid-to-senior (3-8 years experience), not staff/principal level
- [ ] Every question passes: "would a senior engineer at a typical company actually encounter this?"

**List every violating question number with explanation.**

### 10. Coverage Completeness

- [ ] Extract every distinct concept from the summary bullet list
- [ ] Verify each concept is covered by at least one question
- [ ] List any orphan concepts (mentioned in summary but not covered by any question)
- [ ] List any questions that cover topics NOT in the summary bullets — these are scope creep violations. The summary bullets are the source of truth and have been carefully reviewed. If a question covers something not in the bullets, flag it for removal.

### 11. Experience-Based Questions

**Note:** `07-system-design/` topics have no experience-based section. `10-behavioral/` topics have no separate experience-based section — all questions ARE experience-based by nature. Skip this check for both.

- [ ] 3-5 experience-based questions exist
- [ ] All start with "Tell me about a time..." or "Describe a time..."
- [ ] Questions are specific to the topic domain (not generic behavioral questions)
- [ ] No overlap with knowledge questions (experience questions test real-world stories, not knowledge recall)

### 12. Progressive Flow & Readability

**Note:** For system-design and behavioral topics, adapt these checks to their structure — system-design should flow through the design interview naturally, behavioral should progress through leadership themes. The standard-specific checks (foundational → conceptual → practical) don't apply.

- [ ] Foundational questions establish the mental model for the entire topic (standard topics only)
- [ ] Conceptual questions build on foundational — no concept appears that wasn't introduced (standard topics only)
- [ ] Practical questions build on conceptual — they apply what was explained (standard topics only)
- [ ] No concept is used in a practical question without being covered conceptually first (standard topics only)
- [ ] Related questions are adjacent (not scattered across sections)
- [ ] The file reads like a natural learning progression, not a random list

## Output Format

Output a structured report with this exact format:

```
# Question Review: [File Name]

## Summary
- Total questions: N (X theory, Y practical) — or just N for system-design/behavioral
- Topic type: [type] | Target ratio: X%/Y% | Actual ratio: A%/B% (skip for system-design/behavioral)
- Overall: [PASS / X violations found]

## Violations

### [Check Category Name]

**[PASS]** or:

- **Q[number]**: [violation description] → [suggested fix]
- **Q[number]**: [violation description] → [suggested fix]

### [Next Category]
...

## Orphan Concepts
[List any summary concepts not covered, or "None — all concepts covered"]

## Suggestions
[Optional section — broader quality improvements that aren't strict violations but would improve the file. Keep this short — max 3-5 bullet points. Skip this section entirely if there's nothing worth suggesting.]
```

## Output File

Write the report as a sibling file next to the questions file, using `.violation-report.md` as the suffix. This keeps the report next to the file it reviews and makes it easy to find and delete after fixes.

**Pattern**: `{folder}/{file}.violation-report.md`

Replace the `.md` extension with `.violation-report.md`:

**Examples**:

- `04-production/02-prometheus-and-grafana.md` → `04-production/02-prometheus-and-grafana.violation-report.md`
- `01-core-stack/01-nodejs.md` → `01-core-stack/01-nodejs.violation-report.md`
- `03-infrastructure/02-kubernetes.md` → `03-infrastructure/02-kubernetes.violation-report.md`

This allows running the skill on multiple files in parallel — each produces its own named report file next to the source.

After writing the file, output a short summary to the conversation: the file path, total violation count, and a one-line verdict (PASS or the top 2-3 issue categories).

## Important Rules

- Be ruthless. The goal is to catch every issue before answers are written.
- Every violation MUST include a concrete fix suggestion — not just "this is wrong" but "change it to this."
- For question rewrites, write the full replacement question text.
- If a check passes, explicitly mark it as **[PASS]** — don't skip it.
- Do NOT rewrite or modify the questions file. Only write the report to the temp file.
- Do NOT generate answers. You are reviewing questions only.
- Compare quality against the Kubernetes reference file — if a question wouldn't meet that bar, flag it.
