# Interview Espresso

A smaller, focused version of [interviews](https://github.com/YahiaElTai/interviews) — same depth, narrower scope. Used the same way as the main repo: interview prep, day-to-day knowledge base, and quick reference for brushing up on topics.

The main repo is the comprehensive gold standard (49 files, 20-40 questions each). This repo applies the 80/20 principle: ~60-70% of the most important concepts per topic, with full-quality answers. Every question earns its place by being relevant to senior backend interviews, foundational for day-to-day work, or commonly asked by real companies. We sacrifice breadth, not depth.

## The 5 Pillars

Same as the main repo — every answer follows all five:

1. **Completeness** — covers everything important within the selected scope
2. **Clarity** — direct, plain language
3. **Conciseness** — minimum words needed
4. **Accuracy** — current, verified
5. **Practicality** — real-world examples, code, tradeoffs

## How It Was Made

Each file was distilled from the main repo by:
1. Trimming summary bullets to the essential ~70-80% (cutting niche, rarely-asked, and deep-specialist content)
2. Dropping questions whose topics were cut from the summary
3. Keeping answers at full quality — same depth as the main repo

## File Format

Same as the main repo:
- **Summary** — trimmed bullet list of topics covered
- **Knowledge Questions** — collapsible Q&A
- **Experience-Based Questions** — scenario questions at the bottom

## Structure

### Core Stack
| # | File | Description |
|---|------|-------------|
| 01 | [Node.js](01-core-stack/01-nodejs.md) | Event loop, streams, worker threads, memory, core APIs |
| 02 | [TypeScript](01-core-stack/02-typescript.md) | Generics, conditional types, utility types, type guards |
| 03 | [API Design](01-core-stack/03-api-design.md) | REST, GraphQL, gRPC, SSE |
| 04 | [Testing](01-core-stack/04-testing.md) | Test strategies, CI testing, contract testing |
| 05 | [NestJS](01-core-stack/05-nestjs.md) | Modules, DI, guards, interceptors, Prisma/TypeORM |

### Data
| # | File | Description |
|---|------|-------------|
| 01 | [Databases](02-data/01-databases.md) | SQL vs NoSQL, indexing, transactions, replication |
| 02 | [PostgreSQL](02-data/02-postgresql.md) | JSONB, CTEs, partitioning, VACUUM, replication |
| 03 | [Redis](02-data/03-redis.md) | Data structures, caching, pub/sub, distributed locks |

### Infrastructure
| # | File | Description |
|---|------|-------------|
| 01 | [Docker & Containers](03-infrastructure/01-docker-and-containers.md) | Dockerfiles, multi-stage builds, networking, security |
| 02 | [Kubernetes](03-infrastructure/02-kubernetes.md) | Pods, Services, Ingress, networking, autoscaling |
| 03 | [Web Servers & Reverse Proxies](03-infrastructure/03-web-servers-and-reverse-proxies.md) | Nginx, load balancing, SSL termination |
| 04 | [CI/CD](03-infrastructure/04-ci-cd.md) | Pipeline design, deployments, rollbacks, GitOps |
| 05 | [Cloud — GCP & AWS](03-infrastructure/05-cloud-gcp-and-aws.md) | Compute, storage, networking, IAM, serverless |
| 06 | [Terraform & IaC](03-infrastructure/06-terraform-and-iac.md) | HCL, state, modules, remote backends |
| 07 | [Secret Management](03-infrastructure/07-secret-management.md) | Vault, K8s secrets, rotation strategies |
| 08 | [Authentication & Security](03-infrastructure/08-authentication-and-security.md) | JWT, OAuth 2.0, OIDC, OWASP, CORS, TLS |

### Production
| # | File | Description |
|---|------|-------------|
| 01 | [Observability & Monitoring](04-production/01-observability-and-monitoring.md) | Logs, metrics, traces, alerting, SLIs/SLOs |
| 02 | [Prometheus, Grafana & Loki](04-production/02-prometheus-and-grafana.md) | PLG stack, PromQL, alerting, Loki |
| 03 | [Debugging & Troubleshooting](04-production/03-debugging-and-troubleshooting.md) | Log analysis, memory leaks, flame graphs |
| 04 | [Performance & Optimization](04-production/04-performance-and-optimization.md) | Caching, query optimization, load testing |

### Foundations
| # | File | Description |
|---|------|-------------|
| 01 | [Data Structures & Algorithms](05-foundations/01-data-structures-and-algorithms.md) | Arrays, trees, graphs, Big-O, coding patterns |
| 02 | [Networking & Protocols](05-foundations/02-networking-and-protocols.md) | TCP/IP, HTTP, TLS, DNS, WebSockets |
| 03 | [Operating Systems & Linux](05-foundations/03-operating-systems-and-linux.md) | Processes, threads, memory, I/O models |
| 04 | [Concurrency & Parallelism](05-foundations/04-concurrency-and-parallelism.md) | Race conditions, deadlocks, async patterns |
| 05 | [Git & Version Control](05-foundations/05-git-and-version-control.md) | Rebasing, bisect, reflog, branching models |

### Architecture
| # | File | Description |
|---|------|-------------|
| 01 | [Software Architecture](06-architecture/01-software-architecture.md) | Clean architecture, hexagonal, DDD basics |
| 02 | [Design Patterns & Principles](06-architecture/02-design-patterns-and-principles.md) | SOLID, GoF patterns with TypeScript |
| 03 | [Microservices](06-architecture/03-microservices.md) | Monolith vs micro, sagas, circuit breakers |
| 04 | [Event-Driven Architecture](06-architecture/04-event-driven-architecture.md) | Event sourcing, CQRS, pub/sub, idempotency |
| 05 | [Message Queues & Streaming](06-architecture/05-message-queues-and-streaming.md) | Kafka, RabbitMQ, GCP Pub/Sub, SQS/SNS |
| 06 | [Cloud Design Patterns](06-architecture/06-cloud-design-patterns.md) | Circuit breaker, bulkhead, cache-aside, sidecar |
| 07 | [Scaling & Reliability](06-architecture/07-scaling-and-reliability.md) | Horizontal scaling, sharding, CAP theorem |
| 08 | [Search Engines](06-architecture/08-search-engines.md) | Inverted indexes, Elasticsearch, relevance scoring |

### System Design
| # | File | Description |
|---|------|-------------|
| 01 | [Fundamentals](07-system-design/01-fundamentals.md) | Interview framework, estimation, building blocks |
| 02 | [URL Shortener](07-system-design/02-design-url-shortener.md) | Encoding, caching, redirect flows |
| 05 | [Chat System](07-system-design/05-design-chat-system.md) | WebSockets, delivery guarantees, presence |
| 08 | [News Feed](07-system-design/08-design-news-feed.md) | Fan-out, ranking, celebrity problem |

### Frontend
| # | File | Description |
|---|------|-------------|
| 01 | [React & State Management](08-frontend/01-react-and-state-management.md) | Advanced patterns, Server Components |
| 02 | [Browser & Web Performance](08-frontend/02-browser-and-web-performance.md) | Core Web Vitals, code splitting, SSR |
| 03 | [Frontend Architecture](08-frontend/03-frontend-architecture.md) | Monorepos, micro-frontends, design systems |

### AI
| # | File | Description |
|---|------|-------------|
| 01 | [LLMs & How They Work](09-ai/01-llms-and-how-they-work.md) | Transformers, embeddings, RAG, prompting |
| 02 | [AI Tools for Developers](09-ai/02-ai-tools-for-developers.md) | Claude Code, Copilot, effective prompting |
| 03 | [MCP & AI Integrations](09-ai/03-mcp-and-ai-integrations.md) | MCP architecture, function calling, agents |

### Behavioral
| # | File | Description |
|---|------|-------------|
| 01 | [Leadership & Behavioral](10-behavioral/01-leadership-and-behavioral.md) | STAR method, technical decisions, conflict resolution |

## Progress

44 topic files across 10 sections. Distillation and answer generation in progress.
