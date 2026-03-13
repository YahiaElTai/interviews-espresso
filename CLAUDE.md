# Interview Espresso

A smaller, focused version of the comprehensive [interviews](https://github.com/YahiaElTai/interviews) repo. Same answer quality, ~60-70% of the scope. Used the same way as the main repo: interview prep, day-to-day knowledge base, and quick reference. Every question earns its place by being relevant to senior backend interviews, foundational for daily work, or commonly asked by real companies.

See [README.md](README.md) for the full project description, the 5 Pillars, file format, and topic structure.

## Content Generation Rules

### Question Progression

- **`01-core-stack/`**: Start at mid-level and progress to senior-level depth.
- **`02-data/`**: Start at mid-level and progress to senior-level depth.
- **`03-infrastructure/`**: Start at mid-level and progress to senior-level depth.
- **`04-production/`**: Start at mid-level and progress to senior-level depth.
- **`05-foundations/`**: Start with core concepts and build to advanced application.
- **`06-architecture/`**: Start at mid-level and progress to senior-level depth.
- **`07-system-design/`**: Follow the natural flow of a design interview (requirements → estimation → high-level → deep dive → scaling).
- **`08-frontend/`**: Start at senior level directly — skip basics entirely.
- **`09-ai/`**: Start with conceptual understanding and build to practical application.
- **`10-behavioral/`**: No progression — all questions are senior-level.

### Code Examples

- Use TypeScript/Node.js for backend examples.
- Use React/TypeScript for frontend examples.
- Use HCL for Terraform, YAML for Kubernetes and CI/CD.

## Agents & Skills

This repo uses dedicated **agents** (`.claude/agents/`) to execute **skills** (`.claude/skills/`). Each agent is a specialized subagent with its own system prompt, tool restrictions, and model pinning (Opus). Skills define the rules; agents are the execution layer.

**Always use agents to execute skills** — do not run skills directly in the main conversation. Delegate to the appropriate agent instead. This keeps the main context clean and ensures consistent, high-quality output.

| Agent | What it does |
|---|---|
| `distill-light` | Light trim (~10-15% cut) for core-stack topics (Node.js, TypeScript) — only removes tooling trivia and topics belonging to other interview rounds. Preserves all senior-level depth. |
| `distill` | Standard trim (~30-40% cut) for non-core topics — deletes niche summary bullets and their corresponding questions. |
| `review-summary` | Reviews summary bullets for completeness. Ensures important concepts are covered without going too niche. |
| `fix-summary` | Applies coverage report fixes to a topic file's summary bullets, then deletes the report. |
| `generate-questions` | Generates 20-40 interview questions from a file's summary bullets. |
| `review-questions` | Reviews questions for quality, compliance with guidelines, and structural correctness. Outputs a violation report. |
| `fix-questions` | Applies violation report fixes to a topic file's questions (splits, adds, removes, rewrites), then deletes the report. |
| `apply-checklist` | Cross-references a topic file against `checklists.md` and applies missing coverage — adds summary bullets and questions in one pass. |
| `generate-answers` | Fills in senior-level answers for all questions. Batches of 20, stops after each batch. Resumable — skips already-answered questions. Uses context7 for accuracy. |
| `review-answers` | Reviews answers for quality, compliance with the 5 Pillars. Outputs an answer review report. Uses context7 for accuracy checks. |
| `fix-answers` | Applies answer review report fixes to a topic file's answers, then deletes the report. Uses context7 for accuracy verification. |

**Distillation strategy**: Use `distill-light` for `01-core-stack/` topics (primary tools — interviewers go deep here). Use `distill` for everything else.

**Espresso workflow**: Distill (light or standard) → Generate answers → Review answers → Fix answers.

**Full workflow** (if needed): Review summary → Fix summary → Generate questions → Review questions → Fix questions → Apply checklist → Generate answers → Review answers → Fix answers.
