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

**The three categories:**

**Inline completion** (Copilot, Codeium) — predicts the next few lines as you type, directly in your editor. Solves the "boilerplate and pattern continuation" problem. You write a function signature, it fills in the body. Low latency, low friction, no context switching. The limitation is that it only sees the current file and a few open tabs — it cannot reason about architecture or multi-file changes.

**Chat-based** (ChatGPT, Copilot Chat, Cursor chat) — conversational interface where you describe what you need and get code or explanations back. Solves the "I need to think through a problem with a knowledgeable partner" problem. You can paste code, ask questions, iterate on solutions. The limitation is that you manually copy-paste context in and code out — the tool does not touch your codebase directly.

**Agentic** (Claude Code, Cursor Composer, Copilot Workspace) — the AI reads your codebase, makes multi-file edits, runs tests, and iterates autonomously. Solves the "implement this feature across the codebase" problem. The limitation is higher latency, higher cost, and the need for careful review of larger changesets.

**Why this evolution happened sequentially:**

The industry could not jump to agents because each stage required solving prerequisite problems and building user trust incrementally. Inline completion proved LLMs could generate useful code at all. Chat proved they could handle complex reasoning and multi-step problems. Only after both were validated — and developers trusted autocomplete, then chatbots — did the infrastructure and trust for autonomous agents make sense. Agents need reliable code generation (proven by completion), reasoning ability (proven by chat), plus tool use, file system access, and execution sandboxing that took additional engineering.

**How they complement each other in practice:**

- **Inline completion** for flow state — writing code where you know what you want and the tool accelerates typing. Function bodies, test cases, repetitive patterns.
- **Chat** for exploration and design — "how should I structure this migration?", "what are the tradeoffs between these two approaches?", understanding unfamiliar code.
- **Agentic** for execution at scale — implementing a feature across multiple files, large refactors, generating tests for an entire module, onboarding to a new codebase.

A senior engineer moves between all three in a single day. The skill is knowing which tool fits the current task — using an agent for a one-liner is wasteful, using autocomplete for a multi-file refactor is painful.

</details>

<details>
<summary>2. How do Claude Code, GitHub Copilot, and Cursor differ in architecture and interaction model -- what core design decisions distinguish each tool (local vs cloud, IDE-embedded vs standalone, inline vs agentic), what are the strengths and limitations that follow from those design choices, and why does understanding the architecture help you predict when each tool will perform well or poorly?</summary>

**GitHub Copilot:**
- **Architecture**: IDE extension (VS Code, JetBrains, etc.) with cloud inference. Inline completions use a smaller, faster model optimized for low-latency suggestions. Copilot Chat uses a larger model (GPT-4 class) for conversational interaction.
- **Strengths**: Lowest friction — lives inside your editor, suggestions appear as you type. Good for pattern continuation, boilerplate, and single-file work. Massive user base means broad language support.
- **Limitations**: Inline completions have limited context (current file + a few open tabs). Chat mode requires manual context management. Not agentic — does not read your codebase, run commands, or make multi-file edits autonomously.

**Cursor:**
- **Architecture**: Fork of VS Code with AI deeply integrated into the editor. Offers inline completion, chat, and Composer (agentic multi-file editing). Routes to multiple model providers (Claude, GPT-4, etc.). Key design decision: the editor itself is the AI interface, not a plugin bolted on.
- **Strengths**: Best IDE integration — codebase indexing for semantic search, automatic context selection, inline diff review for AI changes. Composer mode enables multi-file edits with a review flow. `.cursorrules` files let you customize behavior per project.
- **Limitations**: Tied to the Cursor editor (VS Code fork) — if your team uses JetBrains or Neovim, it is not an option. Context window still has limits, and the automatic context selection can miss relevant files or include irrelevant ones.

**Claude Code:**
- **Architecture**: Standalone CLI tool (terminal-based, not IDE-embedded). Fully agentic — reads files, writes files, runs shell commands, executes tests, iterates on failures. Uses Claude models with large context windows (200K+ tokens). Operates on the actual filesystem.
- **Strengths**: Deepest codebase understanding because it can read any file, search with grep/glob, and explore the project structure. Best for large-scale changes, onboarding, and tasks that span many files. Project context files (`CLAUDE.md`) give persistent instructions. Works with any editor since it is editor-agnostic.
- **Limitations**: Terminal-based means no inline completions or visual diff review in-editor. Higher latency per interaction (reads files, thinks, writes). Requires trust — it can run commands and modify files. Not ideal for small, quick edits where inline completion is faster.

**Why architecture predicts performance:**

Understanding the architecture tells you what context the tool has access to and how it interacts with your code:

- **Copilot** will struggle with cross-file refactoring because it only sees limited context. It will excel at completing the function you are actively writing.
- **Cursor** will perform well when the codebase is indexed and the relevant files are in its context window, but may produce inconsistent results on very large monorepos where indexing is incomplete.
- **Claude Code** will excel at "implement this feature" tasks because it can explore the codebase, but it will be slower for quick one-line fixes where you just need autocomplete.

The pattern: the more context a tool can access and the more actions it can take, the better it handles complex, multi-file tasks — but the worse it handles simple, fast, single-line tasks where latency matters more than reasoning.

</details>

## Conceptual Depth

<details>
<summary>3. Why is AI-generated code often more dangerous than obviously broken code -- what are the most common failure modes a senior engineer should watch for during review (hallucinated APIs, outdated patterns, subtle security issues, plausible-but-wrong logic), why does each happen given how LLMs generate code, and what systematic review practices catch these issues before they reach production?</summary>

AI-generated code is dangerous precisely because it looks correct. Broken code fails loudly — it does not compile, tests fail, linters catch it. AI code compiles, passes superficial review, and hides subtle issues under a veneer of competence. This is the "uncanny valley" problem: the code is close enough to correct that reviewers lower their guard.

**Common failure modes and why they happen:**

**Hallucinated APIs** — The model generates calls to methods or options that do not exist. This happens because LLMs predict tokens based on patterns in training data. If a method "should" exist based on naming conventions, the model will confidently generate it. Example: calling `response.json({ status: 'ok' })` on a library that uses `response.send()` instead.

**Outdated patterns** — The model uses deprecated APIs, old library versions, or patterns that were best practice 2-3 years ago. Training data has a cutoff, and popular patterns from older StackOverflow answers are overrepresented. Example: using `new Buffer()` instead of `Buffer.from()`, or callback-style Node.js APIs instead of promise-based ones.

**Subtle security issues** — The model generates code that works but is insecure: SQL concatenation instead of parameterized queries, missing input validation, overly permissive CORS, secrets in client-accessible code. LLMs optimize for "code that accomplishes the stated goal," not "code that is secure against unstated threats."

**Plausible-but-wrong logic** — The trickiest failure mode. The code handles the happy path correctly but has wrong boundary conditions, incorrect error handling, or subtle race conditions. The model generates what looks like correct logic based on pattern matching, but it does not truly reason about invariants. Example: an off-by-one in pagination, a retry loop that does not back off, or an async operation missing proper error propagation.

**Confident but incorrect assumptions** — The model assumes a data shape, environment variable, or service behavior that does not match your actual system. It generates code that would work in a generic setup but fails in your specific architecture.

**Systematic review practices:**

1. **Compile and run it** — Never approve AI code you have not executed. Run tests, hit the endpoint, verify behavior.
2. **Check every import and API call** — Verify that every method, option, and API actually exists in the version you are using. This is the fastest way to catch hallucinations.
3. **Review with extra skepticism on edge cases** — AI code tends to handle the happy path well. Specifically check: what happens on empty input, null values, concurrent access, network failures, and boundary conditions.
4. **Security-specific pass** — Separately scan for injection vulnerabilities, missing auth checks, exposed secrets, and overly permissive configurations.
5. **Read it like you wrote it** — The biggest mistake is reviewing AI code as "someone else's code you're approving." Review it as if YOU wrote it and YOU are responsible for it in production.

</details>

<details>
<summary>4. Why does context window management matter so much for AI tool effectiveness — how does the amount and quality of context you provide affect output quality, what strategies exist for maximizing useful context (file selection, summarization, scoping), and what happens when you exceed the context window or fill it with irrelevant information?</summary>

Context is the single biggest lever for AI output quality. The same model, same prompt, will produce dramatically different results depending on what context it has access to. This is because LLMs do not "know" your codebase — they can only work with what is in the current conversation window.

**How context quality affects output:**

- **Too little context**: The model hallucinates types, invents API signatures, uses generic patterns that do not match your codebase style. It cannot follow your project's conventions if it has never seen them.
- **Too much irrelevant context**: The model gets confused about which patterns to follow. If you paste 10 unrelated files, the model may pick up patterns from the wrong file. Irrelevant context dilutes the signal and wastes token budget.
- **Right amount of relevant context**: The model follows your existing patterns, uses correct types, matches your error handling style, and produces code that fits naturally into the codebase.

**What happens when you exceed the window or fill it with noise:**

When you exceed the context window, older content gets truncated (in chat) or the request fails (in API calls). The model literally cannot see information that was pushed out. More insidiously, even within the window, models exhibit a "lost in the middle" phenomenon — information at the start and end of the context gets more attention than information buried in the middle.

**Strategies for maximizing useful context:**

**File selection** — Include only the files directly relevant to the task. For implementing a new endpoint: the route file, the service it calls, the types/interfaces, and one existing similar endpoint as a pattern reference. Not the entire `src/` directory.

**Summarization** — For large codebases, provide high-level architecture descriptions rather than raw code. A 20-line summary of "how our auth middleware works" is more useful than pasting 500 lines of auth code when the task is unrelated to auth internals.

**Scoping conversations** — Start new conversations for new tasks. A conversation about "fix the payment bug" that then pivots to "now refactor the user module" carries irrelevant context from the first task into the second. Fresh context for fresh tasks.

**Project-level context files** — Files like `CLAUDE.md` or `.cursorrules` that provide persistent project-level context: architecture overview, coding conventions, common patterns, things to avoid. These give the model baseline understanding without consuming per-conversation token budget.

**Progressive disclosure** — Start with high-level context, let the model ask for what it needs (agentic tools do this naturally), or add detail incrementally. Do not front-load everything "just in case."

**Practical rule of thumb**: If you are getting bad output, before changing your prompt, check whether the model has access to the right context. Nine times out of ten, the problem is missing context, not a bad prompt.

</details>

<details>
<summary>5. How do prompting techniques differ when you're asking an AI tool to generate code vs understand existing code — why does the same tool perform dramatically differently with different prompt structures, when should you use structured prompts (constraints, examples, format specifications) vs open-ended exploratory prompts, and what makes a prompt effective for each use case?</summary>

**Code generation prompts — structured and constrained:**

When generating code, the model needs to produce a specific artifact. Ambiguity in the prompt leads to ambiguity in the output. Effective generation prompts include:

- **What** you want built (endpoint, function, module)
- **Constraints** (framework, patterns to follow, error handling approach)
- **Input/output examples** or type signatures
- **Anti-patterns** to avoid ("don't use callbacks, use async/await")
- **A reference** — an existing similar piece of code to follow as a pattern

Example of an effective generation prompt:
```
Add a POST /api/orders endpoint to src/routes/orders.ts.
Follow the same pattern as the existing POST /api/products endpoint.
Use Zod for request validation. Return 201 on success, 400 on validation failure.
The order service is at src/services/order.service.ts — use its createOrder method.
Include error handling matching our existing middleware pattern.
```

**Code understanding prompts — open-ended and exploratory:**

When understanding code, you want the model to explain, not produce. Effective understanding prompts are:

- **Open-ended** — "Explain how this middleware chain works" rather than "List the middleware functions"
- **Focused on WHY** — "Why does this use a WeakMap instead of a Map?" rather than "What data structure is this?"
- **Layered** — Start broad ("What does this module do?"), then drill down ("How does the caching layer handle invalidation?")

**Why the same tool performs differently:**

LLMs are token predictors. A structured prompt with constraints and examples narrows the prediction space — the model has fewer "reasonable" next tokens to choose from, so it converges on the right answer faster. An open-ended prompt broadens the space, which is good for exploration but bad for precise generation.

Think of it like giving directions: "Drive to 123 Main Street" (structured) vs. "Tell me about the neighborhoods in this city" (exploratory). Both are valid, but using the wrong type for your goal gives poor results.

**When to use which:**

| Situation | Prompt style | Why |
|---|---|---|
| Generating a new function/endpoint | Structured with constraints and examples | Reduces ambiguity, matches project patterns |
| Understanding unfamiliar code | Open-ended, then drill down | You do not know what you do not know yet |
| Debugging | Mix — describe the symptom (open), then ask for specific causes (structured) | Start broad, narrow as you learn |
| Refactoring | Structured with before/after expectations | The model needs to know what "better" means |
| Design discussion | Open-ended with tradeoff framing | You want options, not a single answer |

**What makes any prompt effective:**

1. **Specificity** — "Fix this code" is terrible. "This function throws on empty arrays — handle that case by returning an empty result" is effective.
2. **Context** — The prompt references actual files, types, and patterns in the codebase.
3. **Intent** — The model knows WHY you want this, not just WHAT you want. "We need this for audit logging" changes the output vs. "add a logging call."

</details>

<details>
<summary>6. Why do single-shot prompts fail for complex engineering tasks, and how does iterative refinement change the quality of AI output — what does an effective refinement loop look like (initial generation, critique, constraint tightening, edge case injection), when should you abandon refinement and write the code yourself, and how do you recognize diminishing returns?</summary>

**Why single-shot fails for complex tasks:**

A complex engineering task has too many requirements, constraints, and edge cases to express in a single prompt. Even if you could write a perfect prompt, the model would need to hold all constraints simultaneously while generating code — and it will inevitably drop some. Single-shot works for isolated functions. It breaks down when the task involves multiple files, error handling, validation, tests, and integration with existing code.

**The conceptual framework for iterative refinement:**

Effective refinement follows a predictable pattern of narrowing scope:

1. **Initial generation (broad strokes)** — Give the high-level task with key constraints. Accept that the first output is a rough draft covering 60-70% of requirements. Deliberately leave out secondary requirements to keep the model focused on structure.
2. **Critique and correct** — Review for structural issues. Be specific about what is wrong ("validation is missing, use Zod") rather than vague ("fix it"). The model improves most when corrections reference concrete patterns in your codebase.
3. **Constraint tightening** — Add the requirements you deliberately held back: edge case handling, authorization, concurrency, integration with other services. Each round narrows the gap between "works" and "production-ready."
4. **Edge case injection** — Push the model to handle failure modes: transaction rollbacks, concurrent requests, malformed input. This is where the model's output becomes robust.

See Q16 for a full practical walkthrough of this loop applied to a real endpoint.

**When to abandon refinement and write it yourself:**

- **After 3-4 rounds on the same issue** — If the model keeps getting the same thing wrong after clear corrections, it is likely hitting a reasoning limitation. Write it yourself.
- **When corrections take longer than writing** — If each refinement prompt takes you 5 minutes to write and the model is only getting 60% of the way there, you would be faster writing code directly.
- **When the model is fighting your architecture** — If it keeps reverting to patterns that do not match your codebase despite explicit instructions, the context mismatch is too large.
- **When you cannot explain what is wrong** — If the output "feels wrong" but you cannot articulate specific issues, the problem might be in your understanding, not the model. Step back and think before prompting more.

**Recognizing diminishing returns:**

Each refinement round should produce noticeably smaller improvements. You have hit diminishing returns when: you are making cosmetic changes (variable names, formatting) rather than structural ones, the model starts introducing new bugs while fixing the ones you pointed out, or you find yourself re-explaining constraints from earlier rounds (the model "forgot" them).

**Practical heuristic:** If the model gets 80% right in the first shot and 95% after one round of refinement, the last 5% is almost always faster to fix by hand than to prompt for.

</details>

<details>
<summary>7. When should you deliberately NOT use AI tools — why are security-critical code paths, complex business rules with subtle edge cases, and novel algorithms poor candidates for AI generation, what specific risks arise when teams ignore these boundaries, and how do you draw the line between "AI-assisted" and "AI-generated" for these sensitive areas?</summary>

**When to avoid AI generation:**

**Security-critical code paths** — Authentication flows, authorization checks, cryptographic operations, input sanitization, token validation. As covered in Q3, AI-generated security code often passes review because it looks correct — but LLMs generate "plausible security code" with subtle flaws: timing-safe comparison replaced with `===`, missing CSRF protection, JWT validation that skips expiry checks, bcrypt rounds set too low. Security code must be written by someone who understands the threat model, not generated by a tool that understands patterns.

**Complex business rules with subtle edge cases** — Tax calculations, financial transactions, regulatory compliance logic, billing with proration and refunds. These domains have edge cases that are not documented in public code and not represented in training data. An LLM will generate code that handles the common case but misses the edge case where a leap year interacts with a billing cycle, or where a currency conversion requires specific rounding rules. These bugs are expensive — they result in wrong charges, compliance violations, or financial discrepancies.

**Novel algorithms** — Anything where your specific problem does not have a well-known pattern in public codebases. LLMs remix existing patterns. If you need a custom scheduling algorithm for your specific constraints, the model will generate something that looks like a scheduler but does not actually solve your problem. It will confidently produce code that "seems right" based on similar-looking problems.

**Specific risks when teams ignore these boundaries:**

- **False confidence** — The team believes the code is correct because "the AI generated it and it passes tests." But the tests were also AI-generated and test the happy path.
- **Diffusion of responsibility** — Nobody fully owns the logic because it was "AI-generated." When a bug surfaces, the debugging is harder because nobody deeply understands the code.
- **Compounding errors** — AI generates the implementation, AI generates the tests, AI generates the documentation. If the initial implementation has a subtle flaw, the tests and docs will be consistent with the flaw, making it nearly invisible.

**Drawing the line — "AI-assisted" vs "AI-generated":**

**AI-generated** = the AI wrote it, you reviewed it. Appropriate for boilerplate, CRUD, test scaffolding, documentation, utilities, and standard patterns.

**AI-assisted** = you wrote it, the AI helped. Appropriate for security, business logic, and novel algorithms. Use AI to:
- Explain existing security patterns you are adapting
- Review your implementation for issues you might have missed
- Generate test cases (but verify them manually)
- Suggest edge cases to consider
- Write the boilerplate around the critical logic (route setup, middleware wiring) while you write the core logic by hand

**Practical rule**: If the code going wrong would cost money, lose data, or create a security breach, a human should write the core logic and use AI only as a reviewer and assistant — not as the author.

</details>

<details>
<summary>8. What does AI dependency look like in individual developers and teams — what are the warning signs that someone is over-relying on AI tools (declining ability to write code without them, accepting output without understanding it, reduced debugging skills), why is this particularly dangerous for junior-to-mid developers, and what practices prevent over-reliance while still capturing productivity gains?</summary>

**Warning signs in individual developers:**

- **Cannot write code without the tool** — When the AI is down or unavailable, productivity drops to near zero. The developer stares at an empty file without knowing where to start.
- **Accepting without understanding** — Committing AI-generated code they cannot explain line by line. If asked "why did you use this approach?", the answer is "that's what the AI suggested."
- **Declining debugging skills** — When something breaks, the first instinct is to paste the error into AI instead of reading the stack trace, checking logs, or reasoning about the code. Debugging becomes "prompt until it works" rather than "understand why it is broken."
- **Prompt-first, copy-paste-pray workflow** — Reaching for the AI tool before thinking about the problem, generating code, pasting it, running tests, and if they fail, asking the AI to fix it — no mental model formed at any step.

**Warning signs in teams:**

- Code reviews become rubber stamps — "the AI wrote it and the tests pass" replaces actual review, and nobody deeply understands any part of the codebase.
- Onboarding depends entirely on AI — new team members cannot navigate the codebase without an AI tool because documentation and code comments were AI-generated and no human has a deep mental model.

**Why this is particularly dangerous for junior-to-mid developers:**

This is the stage where engineers build foundational skills: reading code, debugging, understanding patterns, developing architectural intuition. These skills come from struggling with problems, not from having them instantly solved. A junior developer who uses AI to skip the struggle never builds the mental models that separate senior engineers from everyone else.

The trap is that AI makes junior developers look more productive in the short term — they ship features faster. But they are building on a hollow foundation. When they face a problem the AI cannot solve (production debugging, architecture decisions, novel requirements), they lack the skills to handle it. They hit a ceiling they cannot see coming.

**Practices that prevent over-reliance while capturing productivity:**

1. **Think first, prompt second** — Before asking the AI anything, spend 5 minutes sketching the solution mentally. What files need to change? What is the data flow? What edge cases exist? Then use the AI to execute faster, not to think for you.

2. **Explain what you committed** — Make it a habit: before opening a PR, you should be able to explain every line of AI-generated code. If you cannot, you do not understand it well enough to ship it.

3. **Regular AI-free sessions** — Deliberately work without AI tools periodically. Not as punishment, but as practice — like a musician practicing scales without sheet music.

4. **Debug manually first** — When something breaks, read the error, check the logs, form a hypothesis. Only bring in AI after you have spent at least 10-15 minutes reasoning about it yourself.

5. **Use AI for acceleration, not replacement** — The right mental model: "I know how to do this, and AI helps me do it faster." Not: "I do not know how to do this, so I will ask AI."

6. **Code review AI output rigorously** — Teams should review AI-generated code with the same (or more) scrutiny as human-written code. "The AI wrote it" is not a defense in review.

</details>

<details>
<summary>9. When an AI tool produces poor output, how do you diagnose whether the problem is insufficient context, bad prompting, or a fundamental capability limitation — what does each failure mode look like in practice, why does this diagnosis matter (because the fix is completely different for each), and what systematic approach helps you quickly identify which factor is the bottleneck?</summary>

**The three failure modes and their signatures:**

**Insufficient context** — The output is structurally reasonable but factually wrong about YOUR codebase:
- Uses wrong type names, method signatures, or API patterns
- Invents services or modules that do not exist
- Follows generic patterns instead of your project's conventions
- Gets the architecture wrong (assumes REST when you use GraphQL, assumes SQL when you use MongoDB)

**Bad prompting** — The output is reasonable but does not match what you actually wanted:
- Solves the wrong problem (implements a GET when you wanted a POST)
- Handles the happy path but misses edge cases you care about
- Uses an approach you explicitly do not want
- Produces output in the wrong format (full file when you wanted a function, test when you wanted implementation)

**Capability limitation** — The output is fundamentally flawed in a way that more context or better prompting would not fix:
- Circular logic or contradictory code
- Cannot maintain consistency across a long output (early parts contradict later parts)
- Fails at complex multi-step reasoning (calculating correct state transitions, getting concurrent logic right)
- Keeps making the same mistake after explicit correction
- The task requires reasoning about something not well-represented in training data

**Why the diagnosis matters — the fixes are completely different:**

| Failure mode | Fix |
|---|---|
| Insufficient context | Add the missing files, types, or architecture description |
| Bad prompting | Restructure the prompt — add constraints, examples, or explicit requirements |
| Capability limitation | Stop prompting. Write the code yourself. No amount of context or prompt engineering will fix a reasoning gap. |

Applying the wrong fix wastes time. Adding more context when the problem is bad prompting just dilutes the signal. Refining the prompt when the problem is missing context just generates more confidently wrong output. And iterating on either when the problem is a capability limitation is an infinite loop of frustration.

**Systematic diagnostic approach:**

**Step 1 — Check context first (fastest to rule out):**
Does the model have access to the relevant files, types, and patterns? If you are using a chat tool, did you paste the right files? If you are using an agentic tool, did it read the right files? Quick test: ask the model to describe the current structure of the file or module you want changed. If it gets this wrong, it is a context problem.

**Step 2 — Check prompting (next fastest):**
Re-read your prompt. Is the ask clear and unambiguous? Would a human colleague produce the right output from this prompt? Try rephrasing the same request with explicit constraints, an example of the desired output, or a reference to existing similar code. If the output improves significantly, it was a prompting problem.

**Step 3 — Test for capability limits (last resort):**
Give the model perfect context (paste the exact files) and a clear prompt (explicit requirements with examples). If it still fails, try a simpler version of the same task. If it handles the simple version but fails the complex one, you have found the capability boundary. Common limits: complex state machines, intricate concurrent logic, multi-step mathematical reasoning, maintaining consistency across very long outputs.

</details>

<details>
<summary>10. Why do project-level context files like CLAUDE.md, .cursorrules, and custom instructions exist — what problem do they solve that per-conversation prompting doesn't, what design principles make them effective (specificity, conventions, anti-patterns, architecture context), and what happens to AI tool output quality when a project lacks these files vs has well-crafted ones?</summary>

**The problem they solve:**

Expanding on the context file strategy mentioned in Q4, per-conversation prompting fails for consistent, project-wide AI usage because:
- **Repetition** — You repeat the same instructions ("use Zod for validation, AppError for errors, repository pattern for data access") across dozens of conversations
- **Inconsistency** — You forget different constraints in different conversations, leading to inconsistent AI output
- **Onboarding** — New team members do not know what instructions to give the AI
- **Drift** — As the project evolves, scattered per-conversation instructions become stale

Project-level context files solve all of these: one source of truth, committed to the repo, versioned, reviewed, and automatically loaded by the AI tool.

**Design principles for effective context files:**

**Architecture overview** — Without it, the AI cannot reason about where files should go or how services interact, leading to structurally wrong output. Not the full codebase documentation, but enough: "This is a NestJS monorepo with 3 services. Service A handles ingestion, Service B handles querying, they communicate via a shared PostgreSQL database."

**Coding conventions** — Without explicit conventions, the AI falls back to generic patterns from training data that do not match your codebase. Be specific and actionable: "Use `async/await`, never callbacks. Error handling uses our `AppError` class hierarchy. All database queries go through repository classes, never direct in services."

**Anti-patterns** — The AI's default behavior frequently includes common mistakes; anti-patterns prevent them more effectively than positive instructions alone. "Never use `any` type. Never import from another service's internal modules. Never use `console.log` — use our structured logger."

**Common patterns with examples** — A single concrete example anchors the AI's output more reliably than paragraphs of description. Show one example of the "right way" to write an endpoint, a test, a migration. The AI will follow the pattern.

**Specificity over generality** — Vague instructions ("write clean code") give the AI no actionable constraint and produce no measurable improvement. "Use Zod schemas in `src/schemas/` for request validation" is useful. "Write clean code" is useless.

**What changes with well-crafted context files:**

**Without context files** — The AI generates generic TypeScript that compiles but does not feel like it belongs in your codebase. Wrong patterns, wrong file structure, wrong error handling. Every output needs manual adjustment.

**With well-crafted context files** — The AI generates code that matches your project's style, uses the right patterns, imports from the right paths, and handles errors the way your codebase handles errors. The output needs review but not rewriting.

The difference is often 30-50% less manual editing after AI generation. Over hundreds of interactions, this compounds significantly.

**Maintenance strategy:**

Context files go stale if nobody updates them. Treat them like documentation — update when conventions change, review during major refactors, keep them concise enough that developers actually read and maintain them.

</details>

<details>
<summary>11. Why are AI tools particularly effective for codebase onboarding — what makes exploring an unfamiliar codebase with an AI tool faster than reading documentation or grepping through code manually, what strategies maximize the value (asking about architecture first, then drilling into specific modules), and what are the risks of forming a mental model of a codebase based on AI explanations that may be wrong?</summary>

**Why AI tools accelerate onboarding:**

Traditional onboarding involves reading documentation (often outdated), grepping through code (slow and requires knowing what to search for), and asking colleagues (limited availability). AI tools — especially agentic ones — combine all three: they read the actual code (always current), can search across the codebase instantly, and are available on demand.

The key advantage is **interactive exploration**. Instead of reading a 50-page architecture doc linearly, you can ask "What happens when an HTTP request hits /api/orders?" and get a traced walkthrough through the actual code: the route handler, middleware chain, service layer, database queries. Follow-up with "What triggers order notifications?" and you get the next piece of the puzzle. This question-driven exploration builds a mental model faster because you are actively constructing understanding rather than passively reading.

**Strategies that maximize onboarding value:**

**Start with architecture (top-down):**
- "What are the main services/modules in this project and how do they relate?"
- "What is the directory structure and what does each top-level folder contain?"
- "How does data flow from an incoming request to the database and back?"

**Then drill into critical paths:**
- "Walk me through the authentication flow — from login request to JWT issuance to authenticated request"
- "How does the payment processing work? Show me the key files involved"
- "Where are the database migrations and what schema does the orders table have?"

**Then focus on your first task:**
- "I need to add a field to the user profile. What files would I need to touch?"
- "Show me an existing endpoint that does something similar to what I need to build"

**Use the codebase as source of truth:**
- "Show me the actual code for the auth middleware, not a summary"
- Ask the tool to cite file paths and line numbers so you can verify

**Risks of AI-based mental models:**

**Plausible but wrong explanations** — The AI may describe how a module "should" work based on its name and structure, but miss critical runtime behavior, configuration-driven logic, or undocumented side effects. It might say "this service sends emails" when it actually just queues them for a separate worker.

**Outdated understanding** — The AI reads the current code, but may not understand recent changes in context. If a module was recently refactored, the AI might describe the new structure but miss that some old patterns are still in transition.

**Missing tribal knowledge** — Every codebase has decisions that only make sense with context: "we use this weird pattern because of a limitation in the old database driver." AI will not know this. It might describe the pattern without explaining why, or worse, suggest it is a mistake.

**Mitigation:** Verify AI explanations against actual code. When the AI says "module X calls service Y," open service Y and confirm. Use AI explanations as hypotheses to test, not facts to memorize. Cross-reference with colleagues on anything architectural — "the AI told me the notification system works like this, is that accurate?"

</details>

<details>
<summary>12. Given a set of engineering tasks -- greenfield feature development, large-scale refactoring, debugging a production issue, and exploring an unfamiliar codebase -- which AI tool would you reach for in each case, why, and what factors should drive a team's choice of primary AI tool vs allowing individual choice?</summary>

**Task-to-tool matching:**

**Greenfield feature development** — Agentic tool (Claude Code, Cursor Composer). Greenfield features span multiple files: route, service, repository, types, tests, migration. An agentic tool can scaffold the entire feature following existing patterns, read related code for consistency, and iterate when you point out issues. Inline completion is too narrow. Chat requires too much copy-paste.

**Large-scale refactoring** — Agentic tool. Renaming a pattern across 50 files, migrating from one library to another, restructuring a module — these are exactly what agentic tools excel at. They can read the codebase, understand the scope, make changes across files, run tests, and fix failures. Doing this with inline completion or chat would be painfully manual.

**Debugging a production issue** — Start with your own skills and tools (logs, metrics, stack traces), then bring in chat or agentic for specific analysis. AI tools are useful for: "Here is the stack trace and the relevant code — what could cause this?" or "What race conditions could exist in this async flow?" But AI tools are poor at debugging that requires understanding runtime state, infrastructure, or deployment specifics. Do not paste a production error into AI as your first step — read the error, check dashboards, form a hypothesis first.

**Exploring an unfamiliar codebase** — Agentic tool (as covered in question 11). The ability to read any file, search the codebase, and answer questions interactively makes this the strongest use case for agentic tools. Chat works but requires you to manually paste code. Inline completion is irrelevant here.

**Summary:**

| Task | Best tool type | Why |
|---|---|---|
| Greenfield feature | Agentic | Multi-file generation, pattern following |
| Large-scale refactor | Agentic | Cross-file changes, test-fix iteration |
| Production debugging | Manual first, then chat/agentic for analysis | Requires runtime context AI lacks |
| Codebase exploration | Agentic | Interactive exploration, reads any file |

**Team tool choice — standardize or let individuals choose?**

**Arguments for standardization:** Project-level context files (CLAUDE.md, .cursorrules) only work if the team uses the tool they are written for. Knowledge sharing is easier — "here is how I prompt for X" — when everyone uses the same tool. Cost management and security review are simpler with one tool.

**Arguments for individual choice:** Different tools excel at different tasks. Some developers are more productive in their preferred tool. Forcing a tool switch on a team creates friction.

**Practical approach:** Standardize the primary agentic tool (this is where context files and team workflows matter most). Allow individual choice for inline completion and chat — these are personal productivity tools that do not affect team workflows. Invest in project-level context files for the standardized tool.

</details>

<details>
<summary>13. What are the data privacy risks of using AI coding tools -- why is sending proprietary code to cloud AI APIs a concern, what happens when secrets or credentials appear in prompts, how do local models vs cloud models change the risk profile, and what enterprise data policies should be in place before a team adopts AI tools?</summary>

**Why sending proprietary code to cloud APIs is a concern:**

When you use a cloud AI tool, your code is transmitted to a third-party server for inference. Risks include:

- **Training data inclusion** — Some providers may use your inputs to improve their models, effectively making your proprietary code part of a public model's training data. Most enterprise plans explicitly exclude this, but default/free tiers may not.
- **Data retention** — The provider may log and store your prompts and code for debugging, abuse detection, or other purposes. Even with "no training" guarantees, your code sits on someone else's servers.
- **Transit and storage security** — Data could be intercepted in transit (mitigated by TLS) or accessed through a provider breach.
- **Third-party subprocessors** — The AI provider may use subprocessors (cloud hosting, logging services) that also handle your data.

**Secrets and credentials in prompts:**

This is the most acute risk. Developers copy-paste code or config files into AI tools without sanitizing them. Environment variables with API keys, database connection strings with passwords, private keys, and auth tokens end up in AI provider logs. Even if the provider does not train on your data, your secrets are now stored in their systems.

Agentic tools that read files automatically can pick up `.env` files, config files with credentials, or hardcoded secrets. This is why `.gitignore` patterns and AI tool ignore files (like `.claudeignore`) matter.

**Local models vs cloud models:**

**Local models** (Ollama, LM Studio, local Copilot alternatives): Code never leaves your machine. Zero data privacy risk from transmission. Tradeoff: significantly lower quality (smaller models), slower inference, requires beefy hardware.

**Cloud models with enterprise agreements**: Code is transmitted but covered by data processing agreements (DPAs), no-training guarantees, data residency controls, and SOC 2/ISO compliance. This is the practical middle ground most companies use.

**Enterprise data policies that should be in place before adoption:**

1. **Approved tools list** — Which AI tools are approved for use with company code. Free tiers of consumer tools are usually NOT approved.

2. **Data classification policy** — What can and cannot be sent to AI tools. Common framework:
   - Public/open-source code: any tool
   - Internal code: approved enterprise tools only
   - Secrets/credentials: never sent to any AI tool
   - Customer data/PII: never sent to any AI tool

3. **Secret scanning** — Automated scanning that prevents secrets from being included in AI tool context. Git pre-commit hooks, IDE plugins, or AI tool configuration (ignore files).

4. **Enterprise agreements** — DPAs with AI providers that guarantee: no training on your data, data retention limits, data residency, breach notification, right to delete.

5. **Audit and logging** — Ability to audit what data was sent to AI tools, by whom, and when. Important for compliance (SOC 2, GDPR, HIPAA).

6. **Training and awareness** — Developers need to understand what data goes to AI tools and how to sanitize inputs. A policy nobody reads is useless.

</details>

## Practical — Prompting & Generation

<details>
<summary>14. You need to generate meaningful tests for an existing module — walk through the prompting strategy: how do you provide the right context (source code, existing tests, edge cases), what makes the difference between AI-generated tests that are useful vs tests that just assert the current implementation, show example prompts for generating unit tests vs integration tests for a TypeScript service, and explain how you verify the generated tests actually catch real bugs?</summary>

**The key distinction: behavior-based vs implementation-coupled tests:**

AI tools default to generating tests that "assert the current implementation" — they read the code, see what it does, and write tests that pass. These tests are useless because they will pass even if the code is wrong (they are testing what the code does, not what it should do). Useful tests assert expected behavior based on requirements.

**Prompting strategy:**

**Step 1 — Provide the right context:**
- The source code of the module under test
- The interfaces/types it depends on (so the AI knows the contracts)
- One or two existing tests as a pattern reference (testing framework, assertion style, mock setup)
- The requirements or expected behavior (not just the code)

**Step 2 — Frame the prompt around behavior, not implementation:**

Bad prompt:
```
Write tests for this OrderService class.
```

The AI will read the class and write tests that mirror what the code does — including any bugs.

Good prompt for unit tests:
```
Write unit tests for OrderService.createOrder(). Here are the requirements:
- Should create an order with the given items and calculate total price
- Should reject orders with zero items (throw ValidationError)
- Should reject orders where any item has quantity < 1
- Should apply discount codes (10% off for SAVE10, 20% off for SAVE20)
- Should throw NotFoundError if any product ID doesn't exist in the catalog

Use vitest. Mock the OrderRepository and ProductRepository.
Follow the test style in the existing test file I've included.
Test edge cases: empty discount code, expired discount, duplicate items,
maximum quantity limits.
```

Good prompt for integration tests:
```
Write integration tests for the POST /api/orders endpoint.
Test the full request-response cycle using supertest against the real app
(with a test database).

Test cases:
- Valid order creation returns 201 with order ID
- Invalid request body returns 400 with Zod validation errors
- Nonexistent product ID returns 404
- Unauthenticated request returns 401
- Request with valid discount code applies discount to total

Setup: use the test helper in test/helpers.ts to seed the database
with test products before each test. Follow the existing integration
test patterns in test/integration/.
```

**Step 3 — Verify the generated tests actually catch bugs:**

1. **Run the tests** — They should pass against the current correct implementation.
2. **Mutation testing (manual)** — Deliberately break the code and verify the tests fail:
   - Change `>` to `>=` in a boundary check — does a test catch it?
   - Remove validation logic — does a test catch it?
   - Change the discount calculation — does a test catch it?
3. **Check coverage meaningfully** — Not just line coverage, but are the important behavioral paths tested?
4. **Review assertions** — AI tests often have weak assertions. `expect(result).toBeDefined()` is nearly useless. The assertion should check specific values: `expect(result.total).toBe(89.99)`.
5. **Check mocks** — AI-generated mocks sometimes mock away the very behavior you want to test. Verify that mocks are at the right boundary (mock external dependencies, not the thing under test).

**Common pitfalls in AI-generated tests:**
- Testing implementation details (method call order, internal state) instead of behavior
- Overly broad assertions (`toBeTruthy()` instead of specific value checks)
- Missing error path tests — AI tends to test happy paths
- Mocking too much, making the test a tautology

</details>

<details>
<summary>15. Write a project-level context file (CLAUDE.md or .cursorrules) for a real-world TypeScript/Node.js backend project — show the structure and content, explain why each section is there (project architecture, coding conventions, common patterns, things to avoid), demonstrate how specific instructions change AI output quality with before/after examples, and explain the maintenance strategy so the file doesn't become stale?</summary>

**Example CLAUDE.md for a NestJS order management service:**

```markdown
# CLAUDE.md — Order Service

## Architecture

NestJS monorepo with 2 apps:
- `apps/api` — REST API serving the Merchant Center frontend
- `apps/worker` — Bull queue consumer processing async jobs (email, webhooks)

Shared code lives in `libs/`:
- `libs/database` — TypeORM entities, migrations, repository base class
- `libs/common` — DTOs, decorators, error classes, shared utilities

PostgreSQL for persistence. Redis for Bull queues and caching.

## Commands

- `pnpm test` — Run all unit tests (vitest)
- `pnpm test:e2e` — Run e2e tests (requires running database)
- `pnpm lint` — ESLint + Prettier check
- `pnpm migration:generate` — Generate TypeORM migration from entity changes

## Coding Conventions

- **Validation**: Use Zod schemas in the controller layer via ZodValidationPipe.
  Never validate manually in services.
- **Error handling**: Throw AppError subclasses (NotFoundError, ValidationError,
  ConflictError). The global exception filter maps these to HTTP responses.
  Never throw raw Error or HttpException.
- **Database access**: Always go through repository classes in libs/database.
  Never use EntityManager or QueryBuilder directly in services.
- **Async**: Always async/await, never callbacks or raw .then() chains.
- **Logging**: Use the injected LoggerService, never console.log.
  Include correlationId in all log calls.

## Common Patterns

### Adding a new endpoint
1. Add Zod schema in `apps/api/src/schemas/`
2. Add route in the relevant controller
3. Service method handles business logic
4. Repository method handles database access
5. Add unit test for service, e2e test for the endpoint

### Adding a new async job
1. Define job data interface in `libs/common/src/jobs/`
2. Add producer method in the relevant service
3. Add consumer in `apps/worker/src/processors/`
4. Add test for the processor

## Things to Avoid

- Do NOT use `any` type — use `unknown` and narrow
- Do NOT import from `apps/worker` in `apps/api` or vice versa
- Do NOT write raw SQL — use TypeORM query builder
- Do NOT add business logic in controllers — controllers only validate
  and delegate to services
- Do NOT use default exports — always use named exports
```

**Why each section exists:**

- **Architecture** — Gives the AI the structural mental model. Without it, the AI does not know the monorepo layout, which folders exist, or how services communicate.
- **Commands** — So the AI can run tests, lint, and generate migrations when working agentically.
- **Coding conventions** — The highest-value section. Each rule directly prevents a specific AI failure mode.
- **Common patterns** — Step-by-step recipes the AI follows for common tasks, ensuring consistent file placement and structure.
- **Things to avoid** — Prevents the AI's most common wrong choices. Anti-patterns are often more useful than positive instructions because the AI's default behavior frequently includes these mistakes.

**Before/after with specific instructions:**

Without context file, asking "add an endpoint to get an order by ID":
```typescript
// AI generates:
@Get(':id')
async getOrder(@Param('id') id: string) {
  const order = await this.orderRepo.findOne({ where: { id } });
  if (!order) throw new HttpException('Not found', 404);  // Wrong error class
  return order;  // No DTO mapping, leaks entity
}
```

With the context file:
```typescript
// AI generates:
@Get(':id')
async getOrder(@Param('id', ParseUUIDPipe) id: string) { // NestJS built-in — validates UUID format
  const order = await this.orderService.findById(id);  // Delegates to service
  return order;  // Service handles NotFoundError, DTO mapping
}
```

The context file steers the AI toward the correct patterns: delegating to services, using the right error classes, going through the repository layer.

**Maintenance strategy:**

- **Update on convention changes** — When the team changes a pattern, update the context file in the same PR.
- **Review quarterly** — Skim the file once a quarter to remove stale instructions and add new conventions.
- **Keep it concise** — A 50-line context file that developers maintain beats a 500-line file that goes stale. Only include rules that actually affect AI output.
- **Version control it** — Committed to the repo, reviewed in PRs like any other code. The team owns it collectively.

</details>

<details>
<summary>16. Take a complex engineering task — such as adding a new API endpoint with validation, database queries, error handling, and tests — and demonstrate the iterative prompting workflow from start to finish: show the initial prompt, explain why the first output is insufficient, show how you refine through follow-up prompts (adding constraints, fixing errors, handling edge cases), and identify the point where you stop prompting and start editing manually?</summary>

Building on the refinement framework from Q6, here is the full iterative workflow applied to a real task.

**Task: Add a `PATCH /api/orders/:id/cancel` endpoint to an existing NestJS service.**

**Round 1 — Initial prompt:**

```
Add a PATCH /api/orders/:id/cancel endpoint. An order can only be cancelled
if its status is "pending" or "confirmed" — not if it's "shipped" or "delivered".

Here's the existing order entity (src/entities/order.entity.ts),
the order service (src/services/order.service.ts),
and the order controller (src/controllers/order.controller.ts).
```

**First output — what the AI produces:**

The AI generates a controller method, service method, and basic status check. It works for the happy path. But it is insufficient because:

- No Zod schema for the request (it may accept an optional cancellation reason)
- No authorization check — anyone can cancel any order
- No handling of concurrent cancellation (two requests cancel the same order simultaneously)
- Uses `save()` instead of the project's pattern of using a repository method
- No event emission (the worker needs to send cancellation emails)
- No test

**Round 2 — Add missing requirements:**

```
Good start, but missing several things:

1. Add a Zod schema for the request body: { reason?: string }
   (optional cancellation reason, max 500 chars)
2. Add authorization: only the order owner or an admin can cancel.
   Use our existing @RequireAuth() and check order.userId === req.user.id
3. Use the OrderRepository.updateStatus() method instead of direct save()
4. After cancellation, emit an ORDER_CANCELLED event via our EventEmitter
   so the worker picks it up for email notification
5. Add a unit test for the service method covering:
   successful cancel, wrong status, unauthorized user
```

**Second output — improved but still has issues:**

The AI adds all five points. But reviewing the output reveals:

- The optimistic locking / concurrent cancellation is still not handled
- The test mocks are using `jest.fn()` instead of `vi.fn()` (project uses vitest)
- The event payload does not include the cancellation reason

**Round 3 — Fix specific issues:**

```
Two fixes:
1. Handle concurrent cancellation: use optimistic locking. The updateStatus
   repository method should include a WHERE clause checking the current status.
   If zero rows are updated, throw a ConflictError.
2. Tests should use vitest (vi.fn(), vi.spyOn), not jest.
   Follow the test pattern in src/services/__tests__/product.service.test.ts
3. Include the cancellation reason in the ORDER_CANCELLED event payload.
```

**Third output — 95% correct:**

Now the code handles concurrency, uses the right test framework, and includes the reason in events. Remaining issues are minor:

- A variable name does not match project convention
- The error message string could be more descriptive
- The test is missing one edge case (cancelling an already-cancelled order)

**This is where I stop prompting and edit manually.**

Why? Because:
- The remaining fixes are faster to type than to describe in a prompt
- Each fix is 1-2 lines
- Writing a prompt to change a variable name takes longer than changing it
- I want to add the edge case test myself to make sure I understand the behavior

**The pattern:**

| Round | What happens | Time |
|---|---|---|
| 1 | Get the structure right (60-70% complete) | 2 min prompt, 30 sec generation |
| 2 | Add missing requirements (85-90% complete) | 3 min prompt, 30 sec generation |
| 3 | Fix specific issues (95% complete) | 1 min prompt, 30 sec generation |
| Manual | Polish the last 5% | 5 min editing |

Total: ~12 minutes for a feature that would take 30-45 minutes from scratch. The key insight is knowing when to stop — the cost of prompting increases while the value of each fix decreases. Once you are making cosmetic or minor logic edits, your hands are faster than your prompts.

</details>

## Practical — Review & Debugging

<details>
<summary>17. You're joining a new team with a large unfamiliar codebase — walk through the practical workflow of using an AI tool for onboarding: what questions do you ask first to build a mental model, how do you explore the architecture (entry points, data flow, key abstractions), show example prompts for understanding a specific module's purpose and how it connects to the rest of the system, and explain how you verify the AI's explanations are accurate rather than plausible-sounding hallucinations?</summary>

**Phase 1 — Build the big picture (first 1-2 hours):**

Start with structural questions using an agentic tool that can read the filesystem:

```
Read the project structure and give me a high-level architecture overview:
- What are the main services/apps?
- What framework and language is this?
- What's the directory structure convention?
- How do the main components relate to each other?
```

```
What are the entry points to this application? Show me where HTTP
requests come in, where background jobs are processed, and where
scheduled tasks run.
```

```
What databases and external services does this project depend on?
Check docker-compose files, environment variables, and configuration files.
```

**Phase 2 — Trace critical paths (next 2-3 hours):**

Pick the most important user-facing flows and trace them end-to-end:

```
Walk me through what happens when a user creates an order:
start from the HTTP request hitting the API and trace through
every layer — controller, validation, service, database operations,
any async jobs that get triggered. Show me the actual file paths
and key functions involved.
```

```
How does authentication work in this project? Trace from login
request to JWT issuance to how subsequent requests are authenticated.
What middleware is involved?
```

**Phase 3 — Understand specific modules for your first task:**

```
I need to work on the notification module in src/notifications/.
Explain:
- What does this module do?
- What are its public interfaces (what do other modules call)?
- What external services does it depend on?
- What are the key files and their responsibilities?
- Are there any tests I should look at to understand expected behavior?
```

```
Show me how the notification module connects to the order module.
When an order status changes, how does the notification get triggered?
Show me the actual event/function chain.
```

**Phase 4 — Understand patterns and conventions:**

```
Looking at 3-4 existing endpoints, what patterns does this codebase follow?
How is validation done? Error handling? Database access?
What testing patterns are used?
```

**Verification strategy — do not trust, verify:**

1. **Spot-check file paths** — When the AI says "the auth middleware is in `src/middleware/auth.ts`," open that file and confirm it exists and does what the AI described.

2. **Trace code yourself** — After the AI explains a flow, manually follow the call chain in at least one critical path. Read the actual code at each step. This builds your own mental model rather than relying on the AI's.

3. **Test with known behavior** — If the AI says "cancelled orders trigger a webhook," check: is there a test for this? Can you see the webhook call in a log or in the code? Trigger it locally and verify.

4. **Cross-reference with team members** — After building your AI-assisted mental model, have a 30-minute conversation with a team member. "Here's my understanding of how X works — is this accurate?" This catches the gaps and wrong assumptions the AI introduced.

5. **Watch for confident hallucination signals:**
   - The AI describes a module's behavior but the description does not match the imports or function signatures in the actual code
   - The AI references files or functions that do not exist
   - The explanation is too clean and generic — real codebases have quirks and legacy patterns that a hallucinated explanation will smooth over

**The mental model:** AI onboarding gives you a rough map of the territory in hours instead of days. But a rough map has errors. You verify the map by actually walking the territory — reading code, running the app, and talking to the team. The AI gets you to 70% understanding fast; the last 30% requires hands-on exploration.

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>18. Tell me about a time an AI tool generated code that looked correct but had a subtle bug or security issue that made it to review or production — how did you discover it, what was the impact, and what review practices did you change as a result?</summary>

**What the interviewer is looking for:**
- Honest acknowledgment that AI tools produce flawed code (not blind trust or blind distrust)
- Ability to describe a specific, concrete situation with technical detail
- Understanding of WHY the AI made the mistake (not just "it was wrong")
- Mature response: changed practices rather than blamed the tool or stopped using it
- Evidence of systematic thinking about code review

**Key points to hit:**
- The specific bug or security issue and why it was non-obvious
- How the AI-generated nature of the code affected the review process (was it reviewed less carefully?)
- The discovery mechanism (test, code review, production monitoring, customer report)
- The concrete changes you made to your workflow afterward

**Suggested structure (STAR):**

1. **Situation** — What you were building, why you used an AI tool for it
2. **Task** — The specific piece of code the AI generated
3. **Action** — How you discovered the bug, what it was, and what you did about it
4. **Result** — What you changed about your review process, what the team learned

**Example outline to personalize:**

"I was using [AI tool] to generate a data migration script that moved user records between database tables. The generated code looked correct — it mapped all fields, handled nulls, and even had a transaction wrapper. But it used a `SELECT *` pattern and did not account for a column that had been added since the original table was created. The migration ran successfully but silently dropped data from the new column for 200 records.

I discovered it during a spot check the next day when a user reported missing data. The root cause was that the AI generated code based on the type interface, which did not yet include the new column.

After that, I changed two things: (1) I now always diff AI-generated database operations against the actual schema, not just the TypeScript types. (2) For any data migration, I run it on a snapshot of production data in staging first and verify row-by-row counts and spot checks before running in production."

</details>

<details>
<summary>19. Describe a time you used an AI tool to significantly speed up a complex engineering task — what was the task, how did you use the tool (prompting strategy, iteration), and where did you have to take over manually? What would the task have looked like without the AI tool?</summary>

**What the interviewer is looking for:**
- Evidence that you use AI tools effectively, not just casually
- Specific prompting strategy and iteration — not "I asked it and it worked"
- Self-awareness about where AI helped and where it fell short
- Realistic assessment of time savings (not "100x faster")
- Understanding of the human-AI collaboration dynamic

**Key points to hit:**
- The complexity of the task (why it was non-trivial)
- Your deliberate prompting strategy (context provided, constraints set, how you iterated)
- The specific point where AI output was not good enough and you took over
- A credible time comparison (with vs without AI)

**Suggested structure:**

1. **The task** — What you were building and why it was complex (multiple files, edge cases, integration points)
2. **The AI workflow** — How you broke the task down, what prompts you used, how you iterated
3. **The handoff point** — Where you stopped using AI and wrote code yourself, and why
4. **The comparison** — Honest estimate of time with AI vs without, and what quality of output each produced

**Example outline to personalize:**

"I needed to add comprehensive test coverage to a service module that had 15 methods and zero tests. Manually, this would have been 2-3 days of work.

I used [AI tool] with an iterative approach: first, I gave it the service file and one existing test file as a pattern reference. I prompted it to generate tests for each method, specifying the behavior I expected (not just 'test this method'). For each batch of tests, I reviewed assertions, ran them, and refined: 'this test is asserting the implementation, not the behavior — rewrite to test that an error is thrown when input is invalid.'

I took over manually for two methods that had complex async behavior with race conditions — the AI kept generating tests that tested the mocked behavior rather than the actual concurrency issue. I wrote those tests by hand because I needed to reason about timing.

Total time: about 6 hours instead of the estimated 2-3 days. The AI generated maybe 70% of the final test code. The other 30% I wrote or heavily edited. Without the AI, the biggest time sink would have been the boilerplate setup — mocks, describe blocks, repetitive assertion patterns — which is exactly what AI handles best."

</details>

<details>
<summary>20. Describe a time you had to debug an issue that an AI tool couldn't help with -- or worse, sent you in the wrong direction -- what was the problem, why did the AI tool fail, and how did you ultimately solve it?</summary>

**What the interviewer is looking for:**
- Evidence that you do not blindly depend on AI tools
- Understanding of AI tool limitations (not just "it was wrong")
- Strong fundamental debugging skills that work without AI
- Ability to recognize when AI is sending you in the wrong direction and course-correct
- Analytical thinking about WHY the tool failed

**Key points to hit:**
- The specific problem and why it was hard
- What the AI suggested and why that was wrong or unhelpful
- How you recognized the AI was not helping (what signals told you to stop)
- The actual debugging approach that solved it (logs, metrics, bisecting, reading source code)
- Why the AI could not help with this specific problem (missing runtime context, infrastructure-specific issue, novel bug)

**Suggested structure:**

1. **The problem** — What was broken, what symptoms you observed
2. **The AI attempt** — What you asked the AI, what it suggested, and why it was wrong
3. **The pivot** — How you realized the AI was not helping and what you did instead
4. **The resolution** — How you actually found and fixed the bug
5. **The lesson** — What this taught you about when AI tools are and are not useful

**Example outline to personalize:**

"We had intermittent 503 errors on one endpoint — maybe 2% of requests, no pattern in the request payload. I pasted the error logs and relevant code into [AI tool]. It suggested three potential causes: a missing null check, a timeout configuration issue, and a connection pool exhaustion scenario. I investigated all three — none were the actual problem.

The AI failed because the root cause was infrastructure-specific: a Kubernetes pod was hitting memory limits and getting OOMKilled, but only under specific request patterns that triggered a memory-intensive code path. The AI had no visibility into pod metrics, memory profiles, or the deployment configuration. It could only reason about the application code, and the application code was correct.

I ultimately solved it by checking Kubernetes events (saw OOMKilled restarts), correlating the timing with the 503 errors, profiling the memory-intensive endpoint locally, and discovering that a large response was being buffered entirely in memory before streaming. The fix was switching to a streaming response.

The lesson: AI tools reason about code. When the bug is in the infrastructure, the runtime environment, or the interaction between your code and the platform it runs on, AI tools are mostly guessing. That is when your fundamentals — reading logs, checking metrics, understanding the platform — matter most."

</details>
