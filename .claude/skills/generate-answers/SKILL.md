---
name: generate-answers
description: Generates senior-level answers for all questions in a topic file, following the 5 pillars. Processes in batches of 10 to maintain quality.
argument-hint: <file-path>
disable-model-invocation: false
---

You are generating senior-level interview answers for a fullstack/backend engineer knowledge base.

## Input

Read the file at path: $ARGUMENTS

This file contains a topic title, summary bullet points, and a list of questions in collapsible `<details>` blocks with `<!-- Answer will be added later -->` placeholders. Your job is to replace each placeholder with a complete, high-quality answer.

## The 5 Pillars

Apply these to every answer you write:

1. **Completeness**: Each answer should fully address the question. Cover all important aspects without leaving gaps. Skip obscure trivia.
2. **Clarity**: Write in direct, plain language — like explaining to a smart colleague. No jargon without explanation, no filler sentences.
3. **Conciseness**: Words are a budget. Use the minimum needed to fully convey the idea. A short, dense answer beats a long, padded one. Match answer length to question complexity — simple questions get short answers, complex questions get longer ones.
4. **Accuracy**: Use current, up-to-date information. Use the context7 MCP tool to verify current APIs, syntax, and best practices for specific libraries and tools. No deprecated patterns or outdated advice.
5. **Practicality**: Ground every answer in real-world engineering. Include code examples (TypeScript/Node.js for backend, React/TypeScript for frontend, HCL for Terraform, YAML for K8s/CI) and configuration snippets where relevant. Explain tradeoffs, not just definitions. Show the WHY, not just the HOW.

## Answer Guidelines

### Depth calibration

- **Simple conceptual questions**: 2-4 sentences. Direct and clear.
- **Comparison/tradeoff questions**: A short intro, then a comparison (use bullet list if it helps clarity), then a practical recommendation of when to use which.
- **Practical/implementation questions**: Brief explanation + code example + any gotchas.
- **Scenario/debugging questions**: Walk through the thought process step by step, as if debugging live.
- **Architecture questions**: High-level approach, key components, tradeoffs considered, and why you'd choose this approach.

### Code examples

- Use TypeScript/Node.js for backend examples.
- Use React/TypeScript for frontend examples.
- Use HCL for Terraform, YAML for Kubernetes and CI/CD.
- Keep code examples focused — show only what's relevant to the question. No boilerplate unless the boilerplate IS the point.
- Add brief inline comments only where the code isn't self-explanatory.

### Cross-reference awareness

- Read ALL existing answers in the file before writing new ones.
- If a concept was already explained in a previous answer, reference it briefly ("as covered above in the event loop question") instead of re-explaining it.
- Do not repeat the same code example or explanation across multiple answers. Each answer should add new knowledge.

### Experience-based questions

- For experience-based questions, provide an answer framework — not a script to memorize.
- Include: what the interviewer is looking for, key points to hit, a suggested structure, and an example outline the reader can personalize with their own experience.

## Batching Rules

**Count the total number of questions** (both knowledge and experience-based) in the file before starting.

**First, check for existing answers.** If the file already has some answers filled in, count how many questions still have `<!-- Answer will be added later -->` or `<!-- Answer framework will be added later -->` placeholders. Read all existing answers to maintain context and consistency, then continue from where the previous run left off.

- **10 or fewer unanswered questions**: Answer all remaining unanswered questions in a single pass. Write the complete file.
- **More than 10 unanswered questions**: Answer the next 10 unanswered questions, write the file with those answers filled in (keeping remaining questions with their placeholders intact), then STOP.

**Always report back** with a summary including:
- File path
- How many questions were answered in this batch
- How many questions remain unanswered
- Total questions in the file
- Whether the file is now complete (all questions answered)

## Output Format

Replace the file content, keeping the structure intact:

- **Do not modify** the title, summary bullet list, or stats line.
- **Do not modify** the question text inside `<summary>` tags.
- **Replace** each `<!-- Answer will be added later -->` or `<!-- Answer framework will be added later -->` with the actual answer.
- Keep the `<details>` / `<summary>` structure exactly as-is.
- Use markdown formatting inside answers (headers, bullet lists, code blocks, tables) for readability.

Example of a completed answer:

````markdown
<details>
<summary>What is the difference between process.nextTick() and setImmediate()?</summary>

`process.nextTick()` fires after the current operation completes but before the event loop continues. `setImmediate()` fires on the next iteration of the event loop (check phase).

```typescript
process.nextTick(() => console.log("nextTick")); // fires first
setImmediate(() => console.log("immediate")); // fires second
```

**When to use which:**

- `nextTick` for ensuring something runs before any I/O — like emitting an event after construction but before the caller can attach listeners.
- `setImmediate` for deferring work to the next event loop cycle without starving I/O.

**Gotcha**: Recursive `nextTick` calls can starve the event loop because they always execute before moving to the next phase.

</details>
````

## Important Rules

- Write ONLY answers — do not add, remove, or modify any questions.
- Do not renumber or reorder questions.
- Do not modify the stats line or section headers.
- Every answer must be self-contained enough to be useful on its own, while avoiding redundancy with other answers in the same file.
- If a question's answer would benefit from a diagram, use ASCII art or a simple text-based diagram.
- If invoked on a file where all questions already have answers (no `<!-- Answer will be added later -->` placeholders remain), do NOT overwrite existing answers. Instead, inform the user: "All questions in this file already have answers. No changes made."
