---
name: generate-questions
description: Generates interview questions for a topic file based on its summary bullet points (source of truth). Use when you need to populate a topic file with questions.
argument-hint: <file-path>
disable-model-invocation: false
---

You are generating questions for a senior fullstack/backend engineer knowledge base. Each file is a **mini-book** — when someone masters every question in a file, they should have 80-90% of the knowledge needed to pass any interview on that topic AND solve real-world problems in production.

## Input

Read the file at path: $ARGUMENTS

This file contains a topic title and a summary as a bulleted list of topics to cover.

**The summary bullet points are the source of truth.** They have been carefully reviewed and verified for completeness. Every bullet represents a concept that MUST be covered by at least one question. Do NOT add questions for concepts that aren't in the summary — if it's not in the bullets, it's been deliberately excluded. Do NOT skip any bullet — if it's there, it matters. Your job is to translate these bullets into high-quality questions, not to expand or reduce the scope.

**Safety check**: If the file already has questions with answers filled in (not just `<!-- Answer will be added later -->` placeholders), warn the main agent before proceeding — regenerating will erase all existing answers.

## Two-Step Generation Process

**You MUST follow this two-step process. Do NOT skip to generating questions directly.**

### Step 1: Generate the Question Plan

Before writing any question text, produce a structured plan. This plan locks in the ratio and section structure BEFORE you start writing questions.

**1a. Classify the topic** — Based on the file path and content, determine the topic type:
- **Practical-heavy** (~30% conceptual / ~70% practical): Kubernetes, Docker, PostgreSQL, Redis, Terraform, CI/CD, Testing, NestJS, Search Engines
- **Balanced** (~50% conceptual / ~50% practical): Node.js, TypeScript, API Design, Databases, Auth/Security, Event-Driven Architecture, Microservices, Message Queues, Browser & Web Performance, Frontend Architecture, Observability, Debugging, Performance, Scaling & Reliability, Web Servers, DSA, Git, Prometheus/Grafana/Loki
- **Conceptual-heavy** (~70% conceptual / ~30% practical): Networking & Protocols, OS & Linux, Concurrency & Parallelism, Software Architecture, Cloud Design Patterns, Design Patterns & Principles, LLMs & How They Work, AI Tools for Developers, MCP & AI Integrations
- **System-design** (`07-system-design/`): No ratio classification. Use design interview sections instead of Foundational/Conceptual/Practical. See Question Progression by Category section.
- **Behavioral** (`10-behavioral/`): No ratio classification. Flat list of questions, no sections. See Question Progression by Category section.

**If the topic isn't listed above**, classify based on: the file path category (`03-infrastructure/` → practical-heavy, `05-foundations/` → conceptual-heavy, `04-production/` → balanced) and whether the summary emphasizes hands-on configuration vs understanding concepts.

**1b. Calculate exact question counts** — Decide on total questions (20-40 knowledge + 3-5 experience-based for standard topics), then calculate the exact number per category using the ratio target. **Foundational questions count toward the conceptual side** of the ratio (they are conceptual in nature). Write this out explicitly:

For **standard topics** (everything except system-design and behavioral):
```
Topic type: [practical-heavy / balanced / conceptual-heavy]
Target ratio: [X% conceptual / Y% practical]
Total knowledge questions: N
- Foundational: 2-3 (counts toward conceptual ratio)
- Conceptual: [exact number]
- Practical: [exact number]
- Conceptual side total (foundational + conceptual): [number] ([percentage]%)
- Practical side total: [number] ([percentage]%)
- Experience-based: 3-5
```

For **system-design and behavioral topics**: Just decide total question count (20-40). No ratio, no category breakdown, no experience-based section.

**1c. Write the outline** — List every question as a one-line summary with its type tag. Group by section.

For **single-tool topics**:
```
## Foundational
1. (foundational) [one-line topic description]
2. (foundational) [one-line topic description]

## Conceptual Depth
3. (conceptual) [one-line topic description]
...

## Practical — [Section Name]
N. (practical) [one-line topic description]
...

## Experience-Based
N. (experience) [one-line topic description]
```

For **multi-tool topics** (see Multi-Tool Topics section):
```
## Foundational
1. (foundational) [overall stack introduction]
2. (foundational) [shared concept across tools]

## ToolA — Concepts
3. (conceptual) [ToolA architecture / why it exists]
4. (conceptual) [ToolA deeper concept]
...

## ToolB — Concepts
N. (conceptual) [ToolB architecture / why it exists]
...

## Practical — [Section Name]
N. (practical) [one-line topic description]
...

## Experience-Based
N. (experience) [one-line topic description]
```

**1d. Self-check** — Count the tagged questions. Verify:
- Foundational count matches plan
- Conceptual count matches plan
- Practical count matches plan
- Experience-based count matches plan
- **Every single summary bullet is covered by at least one question** — list each bullet and the question number(s) that cover it. If any bullet is uncovered, add a question for it.
- **WHY before HOW check** — for each bullet that has both a WHY and HOW dimension, verify there is a conceptual question covering the WHY. If a bullet only has a practical HOW question but no conceptual WHY question, add one.
- **No question covers a concept that isn't in the summary bullets** — if you wrote a question for something not in the bullets, remove it.
- The ratio is within 5% of the target

If the counts don't match, fix the outline before proceeding.

**Output the plan, then immediately proceed to Step 2.** Do not stop or wait for approval — the plan is your internal scaffold to ensure the ratio is correct.

### Step 2: Generate the Full Questions

Expand each outline item into a full question following all the rules below. The question types, counts, and section assignments are already locked in Step 1 — do not deviate from the plan.

Write the complete file content, preserving the existing title and summary bullet list exactly. Use the output format specified below.

## Core Philosophy: WHY > HOW > WHAT

Every question must prioritize in this order:

1. **WHY** — Why does this exist? Why this approach over alternatives? Why would you choose or avoid this? This is the #1 priority.
2. **HOW** — How do you configure, implement, debug, or operate this? Show me concrete output.
3. **WHAT** — What is this thing? The "what" should be embedded naturally in every question, never standalone.

A question that only asks "what is X?" is a wasted question. Even foundational questions must demand the "why" — e.g., "What are the core primitives and **why** does K8s separate them this way instead of just running containers directly?"

## No Shallow Questions

Every single question must have depth. If a question can be answered correctly in one or two sentences, it's too shallow. Test with this rule: **would a senior engineer need more than 30 seconds to give a complete answer?** If not, the question needs more depth, more "why," or should be merged with another question.

## No Overpacked Questions

Each question should focus on ONE concept, scenario, or closely related cluster of ideas. If a question contains 3+ distinct sub-topics joined by "and," it's too big — split it. Test with this rule: **could a senior engineer give a complete answer in under 5 minutes of speaking?** If not, the question is trying to cover too much and should be split into separate questions.

Good: "How does PromQL handle rate calculations — explain rate() vs irate(), why you must use rate() on counters, and how aggregation operators work with label matching?"
(One concept: PromQL rate and aggregation — tightly related)

Bad: "Explain PromQL rate calculations, then show how to configure recording rules, and also explain how Alertmanager routing works"
(Three distinct topics crammed together)

## Progressive Depth Structure

Questions MUST be organized into sections that build on each other. The progression is:

### 1. Foundational (2-3 questions)
The mental model everything else builds on. These establish the "what" and "why" of the topic at a high level. Not basic — foundational. They answer: what are the core concepts, why do they exist, and how do they connect?

### 2. Conceptual Depth (5-12 questions depending on topic type)
Goes deeper into specific areas. Each question builds on the foundation and demands understanding of internals, design decisions, and tradeoffs. Include "when NOT to use" and "common mistakes" where relevant.

**WHY before HOW — the conceptual/practical split rule**: Many summary bullets have both a WHY dimension (why does this exist, what problem does it solve, when to use it, tradeoffs) and a HOW dimension (configure it, show the YAML/code). When a bullet has both:
- The **WHY** MUST be covered by a conceptual question — e.g., "Why does K8s need its own storage abstraction (PV/PVC/StorageClass), how does it differ from Docker volumes, and when should you avoid persistent storage?"
- The **HOW** is covered by a practical question — e.g., "Set up persistent storage for a StatefulSet... show the YAML"
- Do NOT skip the conceptual WHY and jump straight to practical HOW. The reader needs to understand WHY something exists before being asked to configure it.

Some bullets are purely conceptual (e.g., "when to adopt K8s vs alternatives") — these only need a conceptual question. Some bullets are purely practical (e.g., "production debugging: CrashLoopBackOff") — these only need practical questions. But bullets that describe a mechanism or abstraction with both purpose and configuration MUST get the WHY covered conceptually first. This may mean the conceptual question count is higher than the ratio strictly suggests — that's fine. Never sacrifice WHY coverage to hit a ratio target.

### 3-5. Practical sections (total practical questions determined by ratio)

Split practical questions into 2-4 subsections with descriptive subtitles (see "Section Subtitles" below). The typical progression is simpler → complex → debugging, but adapt the split to fit the topic. Common patterns:

- **Configuration & Implementation** — Single-concept config. "Show me the code/config/YAML and explain why each part matters." These assume the reader already understands WHY from the conceptual section — focus on the HOW.
- **Composition & Production** — Combine multiple concepts into production-ready setups. "Set up X with Y and Z working together."
- **Debugging & Troubleshooting** — Real-world failure scenarios. "X is broken — walk through the exact commands/steps to diagnose and fix it." Mirror actual on-call situations.

These are **guidelines, not rigid requirements**. Use 2 subsections for focused topics, 4 for broad ones. Name them based on what the questions actually cover (e.g., K8s uses 4 custom-named practical sections).

**For conceptual-heavy topics**: Collapse the practical sections. Instead of 3 separate practical subsections, use 1-2 combined practical sections (e.g., "Practical — Implementation & Debugging") and expand the Conceptual Depth section to 8-12 questions. The depth comes from understanding, not from config files.

### 6. Experience-Based (3-5 questions)
"Tell me about a time you..." questions specific to the topic domain.

## Multi-Tool Topics

When a file covers multiple distinct tools or systems (e.g., Prometheus + Grafana + Loki, or an ELK stack), do NOT interleave questions across tools in the conceptual section. Instead:

1. **Foundational introduces the overall stack** — why these tools exist together, what each one's role is, and the mental model for how they connect
2. **Split conceptual depth into tool-specific subsections** — e.g., "Prometheus — Concepts", "Grafana — Concepts", "Loki — Concepts". Each subsection gets its own `##` heading
3. **Within each subsection, progress naturally** — start with "why this tool exists / architecture" → deeper internals → tradeoffs. Don't assume prior knowledge of the tool from a different section
4. **Practical sections can group by tool or by workflow** — whichever creates a more natural flow for hands-on work
5. **Debugging section can mix tools** — real incidents cross tool boundaries, so interleaving is natural here

**How to detect**: If the file title contains "&", "+", "and", or lists multiple tool names, treat it as a multi-tool topic. Also check the summary — if it describes distinct tools with their own architectures, it's multi-tool even if the title doesn't make it obvious.

## Real-World Relevance

Every question must pass this filter: **would a senior engineer at a typical company (not Google/Meta scale) actually encounter this?**

- **Prefer libraries over hand-rolling** — Don't ask "implement a job queue from scratch with Redis lists" when everyone uses Bull/BullMQ. Instead ask about configuring and operating the library. The "from scratch" version belongs in a conceptual question about how it works under the hood, not a practical one demanding full implementation.
- **Prefer managed services over raw config** — Don't ask "write a sentinel.conf" when most teams use ElastiCache, Memorystore, or Aiven. Instead ask "what settings matter when configuring a managed Redis instance" or "what do you need to understand about Sentinel even when using a managed service."
- **Skip niche features most teams never touch** — If a feature is used by <10% of teams (e.g., Redis Lua scripting, Streams consumer groups when Kafka exists), either skip it entirely or fold it into a broader conceptual question ("what options exist for X") rather than giving it a dedicated practical question.
- **Complexity ceiling is mid-to-senior** — Target engineers with 3-8 years of experience. If only staff/principal engineers or domain specialists would need this knowledge, it's too deep. The goal is 80-90% coverage of what matters, not 100% coverage of everything possible.

## Question Design Rules

1. **Lead with "why" wherever possible** — "Why does X need Y?" is almost always better than "What is Y?"
2. **Include tradeoffs** — "What are the tradeoffs of X vs Y?" or bake tradeoffs into practical questions
3. **Include "when NOT to"** — "When is X overkill?" / "When should you avoid X?" / "What breaks if you misuse X?"
4. **Practical questions must demand concrete output** — The question itself should make it clear that code, config, YAML, commands, or architecture diagrams are expected. Use phrases like "show the YAML," "write a config," "walk through the exact commands"
5. **Practical questions should explain what breaks** — "...and explain what happens if you skip this" / "...and what breaks in production without it"
6. **Scenario questions should be specific** — Not "how do you debug X?" but "X is doing Y and you see Z — walk through the diagnosis step by step"
7. **No orphan bullets** — Every summary bullet must be covered by at least one question. The summary bullets are the source of truth — they define the scope.
8. **No scope creep** — Do not generate questions for concepts that aren't in the summary bullets. If it's not in the bullets, it was deliberately excluded.
9. **Group related questions** — Questions about related concepts should be adjacent so the reader builds understanding progressively

## Question Progression by Category

Detect the topic category from the file path and adapt the section structure accordingly:

- **`01-core-stack/`, `02-data/`, `03-infrastructure/`, `06-architecture/`**: Use the standard progressive depth structure (Foundational → Conceptual Depth → Practical sections → Debugging). Start foundational at mid-level (not beginner).
- **`04-production/`**: Use the standard structure. Start at mid-level, progress to senior. Heavy emphasis on debugging, tooling, and real incident scenarios.
- **`05-foundations/`**: Use the standard structure. Start with core concepts and build to advanced application. Practical questions should use TypeScript/Node.js examples.
- **`07-system-design/`**: **Use a DIFFERENT section structure** that mirrors a design interview. Do NOT use "Foundational / Conceptual Depth / Practical." Instead use sections like: "Requirements & Estimation" → "High-Level Architecture" → "Component Deep Dives" → "Scaling & Failure Handling". Practical questions should ask for specific database schemas, API designs, and scaling calculations. The WHY-first and no-shallow-questions rules still apply.
- **`08-frontend/`**: Use the standard structure but start at senior level — skip basics entirely. Foundational questions should still exist but assume deep frontend experience.
- **`09-ai/`**: Use the standard structure. Start with conceptual understanding, build to practical application and integration.
- **`10-behavioral/`**: **No section structure needed.** All questions are senior-level. No Foundational/Conceptual/Practical split — just a flat list of numbered behavioral questions followed by no experience-based section (behavioral questions ARE experience-based by nature).

## Output Format (Step 2 only)

Write the complete file content, preserving the existing title and summary bullet list exactly. Use the following structure.

**IMPORTANT**: Do NOT include a Table of Contents. GitHub's `<details>/<summary>` tags collapse the answers, so the section headers and numbered questions provide all the navigation needed. A TOC would be redundant.

**Stats line format:**
- **Standard topics** (fundamentals, backend, infrastructure, production, frontend, AI): `> **N questions** — X theory, Y practical` where theory = foundational + conceptual, practical = practical + experience-based, and N = X + Y.
- **System-design and behavioral topics**: `> **N questions**` — just the total count.

**Single-tool topics:**
```markdown
# [Existing Title]

> **N questions** — X theory, Y practical

[Existing summary bullet list — do not modify]

---

## Foundational

<details>
<summary>1. Question text here?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>3. Question text here?</summary>

<!-- Answer will be added later -->

</details>

## Practical — [Subtitle]

<details>
<summary>N. Question text here</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>N. Tell me about a time you...</summary>

<!-- Answer framework will be added later -->

</details>
```

**Multi-tool topics** — use tool-specific concept sections instead of a single "Conceptual Depth":
```markdown
## Foundational

## ToolA — Concepts

## ToolB — Concepts

## Practical — [Subtitle]

## Practical — Debugging & Troubleshooting

## Experience-Based Questions
```

**System-design topics** — use design interview sections instead of the standard progression:
```markdown
# [Existing Title]

> **N questions**

[Existing summary bullet list — do not modify]

---

## Requirements & Estimation

<details>
<summary>1. Question text here?</summary>

<!-- Answer will be added later -->

</details>

## High-Level Architecture

<details>
<summary>N. Question text here?</summary>

<!-- Answer will be added later -->

</details>

## Component Deep Dives

<details>
<summary>N. Question text here?</summary>

<!-- Answer will be added later -->

</details>

## Scaling & Failure Handling

<details>
<summary>N. Question text here?</summary>

<!-- Answer will be added later -->

</details>
```

Section names are flexible — use whatever fits the specific system design topic. The examples above are a starting point, not a rigid template.

**Behavioral topics** — flat list, no section headers, no experience-based section (all questions ARE experience-based):
```markdown
# [Existing Title]

> **N questions**

[Existing summary bullet list — do not modify]

---

<details>
<summary>1. Question text here?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>2. Question text here?</summary>

<!-- Answer framework will be added later -->

</details>
```

## Section Subtitles

Name the practical subsections based on what makes sense for the topic. Examples:
- K8s: "Single Resource Configuration" → "Multi-Resource Composition" → "Production Scenarios" → "Debugging & Troubleshooting"
- Databases: "Query Writing & Optimization" → "Schema Design & Migrations" → "Production Operations" → "Debugging & Troubleshooting"
- Node.js: "Implementation & Configuration" → "Production Patterns" → "Debugging & Profiling"

Use your judgment — the subtitles should describe what that group of questions covers.

## Code & Config Language

When questions hint at code/config examples in the answer, use the appropriate language for the topic:
- **Backend/fundamentals**: TypeScript / Node.js
- **Frontend**: React / TypeScript
- **Kubernetes/Docker**: YAML
- **Terraform**: HCL
- **CI/CD**: YAML (CircleCI, GitHub Actions)
- **Databases/PostgreSQL**: SQL
- **Cloud**: CLI commands (gcloud, aws)

## Important Rules

- Do NOT write answers — only generate the questions. Answers will be filled in separately.
- Each question gets its own `<details>` block with `<!-- Answer will be added later -->` as placeholder (or `<!-- Answer framework will be added later -->` for experience-based).
- Aim for 20-40 knowledge questions per topic depending on scope. Use your judgment — a broad topic like K8s or databases needs more questions than a focused one like Redis or Git.
- Experience-based questions should be 3-5 per standard topic (system-design has none, behavioral has all).
- **Number every question sequentially** from 1 to N across all sections (e.g., "1. Question text", "2. Question text"). The numbering continues across section boundaries — do not restart at 1 for each section.
- Questions should flow logically within each section — build in complexity.
- The stats line must match the format defined above: `> **N questions** — X theory, Y practical` for standard topics, `> **N questions**` for system-design and behavioral.
- Place a `---` separator after the summary paragraph (before the first section) and before the Experience-Based Questions section (standard topics only — behavioral has no experience section).
