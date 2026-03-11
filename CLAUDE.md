# Interview Espresso

Essential interview prep — distilled from the comprehensive [interviews](https://github.com/YahiaElTai/interviews) repo. Same answer quality, ~70-80% of the scope.

See [README.md](README.md) for the full project description, the 5 Pillars, file format, and topic structure.

## Content Generation Rules

### Question Progression

Same as the main repo:

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

This repo shares agents and skills with the main interviews repo (defined in `.claude/agents/` and `.claude/skills/` there). The key agents used here:

| Agent | What it does |
|---|---|
| `distill` | Trims a topic file to essential ~70-80% scope — removes niche summary bullets and their corresponding questions. |
| `generate-answers` | Fills in senior-level answers for all questions. Batches of 10, resumable. Uses context7 for accuracy. |
| `review-answers` | Reviews answers for quality, compliance with the 5 Pillars. Outputs an answer review report. |
| `fix-answers` | Applies answer review report fixes to a topic file's answers, then deletes the report. |

**Workflow**: Distill (from main repo) → Generate answers → Review answers → Fix answers.
