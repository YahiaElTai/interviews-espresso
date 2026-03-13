# AI Tools for Developers

> **20 questions** — 13 theory, 7 practical

- AI developer tool categories: inline completion, chat-based, agentic
- Claude Code, GitHub Copilot, Cursor — architecture, strengths, and when to use each
- Reviewing AI-generated code: hallucinated APIs, outdated patterns, security vulnerabilities, common mistake patterns
- Context window management: file selection strategies, summarization, scoping conversations, impact of irrelevant context on output quality
- Prompting techniques: code generation vs code understanding, iterative refinement, structured vs open-ended prompts
- When NOT to use AI tools: security-critical logic, complex business rules, novel algorithms
- AI dependency: recognizing and preventing over-reliance in developers and teams
- Data privacy and leakage: risks of sending proprietary code to AI APIs, secrets in prompts, local vs cloud model tradeoffs, enterprise data policies
- AI-assisted test generation: prompting for behavior-based tests vs implementation-coupled tests, providing edge cases, unit vs integration test generation
- Codebase onboarding with AI tools: exploring architecture, understanding data flow, verifying AI explanations against actual code
- Diagnosing poor AI output: context, prompting, or capability limits
- Project-level context files: CLAUDE.md, .cursorrules, custom instructions

---

## Foundational

<details>
<summary>1. What are the three categories of AI developer tools — inline completion, chat-based, and agentic — why did the industry evolve through these stages rather than jumping straight to agents, what problem does each category solve that the others don't, and how do they complement each other in a senior engineer's daily workflow?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How do Claude Code, GitHub Copilot, and Cursor differ in architecture and interaction model -- what core design decisions distinguish each tool (local vs cloud, IDE-embedded vs standalone, inline vs agentic), what are the strengths and limitations that follow from those design choices, and why does understanding the architecture help you predict when each tool will perform well or poorly?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. Why is AI-generated code often more dangerous than obviously broken code -- what are the most common failure modes a senior engineer should watch for during review (hallucinated APIs, outdated patterns, subtle security issues, plausible-but-wrong logic), why does each happen given how LLMs generate code, and what systematic review practices catch these issues before they reach production?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>4. Why does context window management matter so much for AI tool effectiveness — how does the amount and quality of context you provide affect output quality, what strategies exist for maximizing useful context (file selection, summarization, scoping), and what happens when you exceed the context window or fill it with irrelevant information?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How do prompting techniques differ when you're asking an AI tool to generate code vs understand existing code — why does the same tool perform dramatically differently with different prompt structures, when should you use structured prompts (constraints, examples, format specifications) vs open-ended exploratory prompts, and what makes a prompt effective for each use case?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. Why do single-shot prompts fail for complex engineering tasks, and how does iterative refinement change the quality of AI output — what does an effective refinement loop look like (initial generation, critique, constraint tightening, edge case injection), when should you abandon refinement and write the code yourself, and how do you recognize diminishing returns?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. When should you deliberately NOT use AI tools — why are security-critical code paths, complex business rules with subtle edge cases, and novel algorithms poor candidates for AI generation, what specific risks arise when teams ignore these boundaries, and how do you draw the line between "AI-assisted" and "AI-generated" for these sensitive areas?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What does AI dependency look like in individual developers and teams — what are the warning signs that someone is over-relying on AI tools (declining ability to write code without them, accepting output without understanding it, reduced debugging skills), why is this particularly dangerous for junior-to-mid developers, and what practices prevent over-reliance while still capturing productivity gains?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. When an AI tool produces poor output, how do you diagnose whether the problem is insufficient context, bad prompting, or a fundamental capability limitation — what does each failure mode look like in practice, why does this diagnosis matter (because the fix is completely different for each), and what systematic approach helps you quickly identify which factor is the bottleneck?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. Why do project-level context files like CLAUDE.md, .cursorrules, and custom instructions exist — what problem do they solve that per-conversation prompting doesn't, what design principles make them effective (specificity, conventions, anti-patterns, architecture context), and what happens to AI tool output quality when a project lacks these files vs has well-crafted ones?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why are AI tools particularly effective for codebase onboarding — what makes exploring an unfamiliar codebase with an AI tool faster than reading documentation or grepping through code manually, what strategies maximize the value (asking about architecture first, then drilling into specific modules), and what are the risks of forming a mental model of a codebase based on AI explanations that may be wrong?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. Given a set of engineering tasks -- greenfield feature development, large-scale refactoring, debugging a production issue, and exploring an unfamiliar codebase -- which AI tool would you reach for in each case, why, and what factors should drive a team's choice of primary AI tool vs allowing individual choice?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. What are the data privacy risks of using AI coding tools -- why is sending proprietary code to cloud AI APIs a concern, what happens when secrets or credentials appear in prompts, how do local models vs cloud models change the risk profile, and what enterprise data policies should be in place before a team adopts AI tools?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Prompting & Generation

<details>
<summary>14. You need to generate meaningful tests for an existing module — walk through the prompting strategy: how do you provide the right context (source code, existing tests, edge cases), what makes the difference between AI-generated tests that are useful vs tests that just assert the current implementation, show example prompts for generating unit tests vs integration tests for a TypeScript service, and explain how you verify the generated tests actually catch real bugs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. Write a project-level context file (CLAUDE.md or .cursorrules) for a real-world TypeScript/Node.js backend project — show the structure and content, explain why each section is there (project architecture, coding conventions, common patterns, things to avoid), demonstrate how specific instructions change AI output quality with before/after examples, and explain the maintenance strategy so the file doesn't become stale?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. Take a complex engineering task — such as adding a new API endpoint with validation, database queries, error handling, and tests — and demonstrate the iterative prompting workflow from start to finish: show the initial prompt, explain why the first output is insufficient, show how you refine through follow-up prompts (adding constraints, fixing errors, handling edge cases), and identify the point where you stop prompting and start editing manually?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Review & Debugging

<details>
<summary>17. You're joining a new team with a large unfamiliar codebase — walk through the practical workflow of using an AI tool for onboarding: what questions do you ask first to build a mental model, how do you explore the architecture (entry points, data flow, key abstractions), show example prompts for understanding a specific module's purpose and how it connects to the rest of the system, and explain how you verify the AI's explanations are accurate rather than plausible-sounding hallucinations?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time an AI tool generated code that looked correct but had a subtle bug or security issue that made it to review or production — how did you discover it, what was the impact, and what review practices did you change as a result?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>19. Describe a time you used an AI tool to significantly speed up a complex engineering task — what was the task, how did you use the tool (prompting strategy, iteration), and where did you have to take over manually? What would the task have looked like without the AI tool?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>20. Describe a time you had to debug an issue that an AI tool couldn't help with -- or worse, sent you in the wrong direction -- what was the problem, why did the AI tool fail, and how did you ultimately solve it?</summary>

<!-- Answer framework will be added later -->

</details>
