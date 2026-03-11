# MCP & AI Integrations

> **34 questions** — 20 theory, 14 practical

- MCP core primitives: servers, clients, resources, tools, prompts
- MCP transport layer: stdio vs HTTP+SSE — tradeoffs and security
- MCP tool schemas: JSON Schema definitions and tool call accuracy
- MCP server authentication and authorization patterns
- Building MCP servers in TypeScript with the MCP SDK
- Testing and debugging MCP servers: MCP Inspector, integration tests
- MCP deployment: diagnosing transport, auth, and timeout issues
- Function calling in practice: tool-use loop lifecycle, handling tool call results, error propagation, and timeout management in backend services
- AI agents vs simple API calls: ReAct, plan-and-execute architectures
- AI workflow orchestration: low-code platforms vs custom integration pipelines, tradeoffs in flexibility, debugging, and maintainability
- Tool use vs RAG: when to give LLMs live access to systems vs pre-indexed knowledge, combining both, failure modes of each
- RAG backend integration: vector database setup (pgvector, Pinecone), ingestion pipeline architecture, chunking strategies, retrieval API design
- LLM provider abstraction: adapter patterns, SDK wrappers (Vercel AI SDK, LangChain), switching providers without rewriting business logic
- LLM API backend patterns: streaming responses to clients via SSE/WebSockets, error handling, rate limit management, retry with backoff
- Architectural patterns for AI features: sync, async queues, streaming, background enrichment
- Cost and latency tradeoffs: caching LLM responses, token cost estimation, model selection
- Structured output from LLMs: JSON mode, tool-use-as-schema, validation
- Multi-turn conversation management: message history, context window limits, token budgets
- Integration safety: validating tool calls, preventing infinite agent loops, sandboxing tool execution, prompt injection through tool results
- Observability for LLM calls: logging, tracing multi-step agents, metrics and alerts
- Prompt versioning and evaluation: managing prompt changes, testing non-deterministic output
- AI feature rollout: shadow mode, A/B testing, human-in-the-loop fallbacks, gradual rollout strategies for non-deterministic features

---

## Foundational

<details>
<summary>1. What is MCP (Model Context Protocol), what problem does it solve, and how do its five core primitives — servers, clients, resources, tools, and prompts — fit together to give an LLM structured access to external capabilities? Why was a protocol needed instead of every AI tool building its own bespoke integration?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. How does function calling (tool use) work inside an LLM — what is the tool-use loop, how does the model decide to call a tool vs respond directly, what happens between the model emitting a tool call and receiving the result, and why is this loop the foundation for every AI integration pattern (MCP, agents, RAG)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. What distinguishes an AI agent from a simple LLM API call — why would you build an agent that reasons across multiple steps instead of making a single prompt-response call, what are the ReAct and plan-and-execute architectures, and when does the added complexity of an agent not pay off?</summary>

<!-- Answer will be added later -->

</details>

## MCP — Concepts

<details>
<summary>4. MCP supports two transport mechanisms — stdio and HTTP+SSE. Why do two transports exist, what are the tradeoffs between them in terms of latency, security, deployment complexity, and multi-user support, and how does the transport choice affect where and how you can deploy an MCP server?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. MCP tools are defined using JSON Schema — why does the protocol require formal schema definitions for every tool, how do well-designed schemas improve tool call accuracy from the LLM, and what are the common schema design mistakes that cause the model to call tools incorrectly or with bad arguments?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does authentication and authorization work for MCP servers — what patterns exist for securing an MCP server (API keys, OAuth, session tokens), how does auth differ between stdio and HTTP+SSE transports, and what are the risks if an MCP server exposes tools without proper authorization checks?</summary>

<!-- Answer will be added later -->

</details>

## AI Integration — Concepts

<details>
<summary>7. When should you give an LLM access to tools (function calling / MCP) vs use RAG (retrieval-augmented generation) to provide knowledge — what problem does each approach solve, where do they overlap, when would you combine both in the same system, and what are the failure modes unique to each?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. What are the main architectural patterns for adding AI features to a backend — synchronous request-response, async queue-based processing, streaming, and background enrichment? Why does each pattern exist, what determines which one to use for a given feature, and what breaks if you pick the wrong pattern (e.g., synchronous for a slow LLM call)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What are the cost and latency tradeoffs when integrating LLMs into a production backend — how do you estimate token costs for a feature, what strategies exist for caching LLM responses, and how do you decide between a larger more capable model vs a smaller faster one for a given use case?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do you get structured output from an LLM reliably — what are the approaches (JSON mode, tool-use-as-schema, output parsers), why is raw text extraction fragile, and how do you validate and handle cases where the model returns output that doesn't match your expected schema?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. How do you manage multi-turn conversations with an LLM in a backend service — how is message history stored and passed to the API, what happens when the conversation exceeds the context window limit, what strategies exist for truncation and summarization, and how do you set token budgets to balance quality vs cost?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What are the safety risks when an LLM can call tools — how do you validate tool call arguments before execution, what causes infinite agent loops, and what guardrails prevent runaway agent behavior in production?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. How can prompt injection through tool results compromise an agent's behavior — what does this attack look like, why is it different from direct prompt injection, and why is sandboxing tool execution important even when you trust the tool's source code?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. What role do AI workflow automation platforms like n8n play in connecting LLMs to external services — when does a low-code workflow tool make sense vs building a custom integration in code, what are the tradeoffs in flexibility, debugging, and maintainability, and when does low-code become a liability?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. What backend patterns handle LLM API calls in production — how do you stream responses to clients via SSE or WebSockets instead of waiting for the full response, how do you handle LLM provider errors gracefully (timeouts, 429s, 500s), and what does a robust retry-with-backoff strategy look like for LLM APIs that have unpredictable latency?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do you add observability to LLM-powered features — what should you log for each LLM call (tokens, latency, model, prompt version), how do you trace a multi-step agent that makes several tool calls, what metrics and alerts matter for catching regressions in cost, latency, or output quality?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. How do you manage prompt versioning and evaluate prompt changes in production — why is changing a prompt risky even when it seems minor, what strategies exist for versioning prompts, and how do you test and evaluate non-deterministic LLM output to know whether a prompt change improved or degraded quality?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. Why does RAG require a separate ingestion pipeline and vector database instead of just passing documents directly to the LLM — what are the architectural components (chunking, embedding, indexing, retrieval), what tradeoffs exist in chunking strategy and embedding model selection, and what are the common failure modes that cause RAG to return irrelevant or stale results?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Why do teams build an abstraction layer over LLM provider SDKs instead of calling OpenAI or Anthropic directly — what do adapter patterns and SDK wrappers like the Vercel AI SDK or LangChain give you, what are the tradeoffs of each approach (thin wrapper vs full framework), and what breaks when you need to switch providers without an abstraction in place?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. How do you roll out an AI-powered feature safely given that LLM output is non-deterministic — what is shadow mode and how does it let you validate AI behavior before users see it, how do you A/B test a feature where outputs vary on every call, and when do you add human-in-the-loop fallbacks as a safety net?</summary>

<!-- Answer will be added later -->

</details>

## Practical — MCP Implementation

<details>
<summary>21. Build an MCP server in TypeScript using the MCP SDK that exposes at least two tools and one resource — show the project setup, how you define tool schemas, implement tool handlers, register a resource, and configure the transport. Explain the key decisions in the code and what happens if a tool handler throws an error.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>22. How do you test and debug an MCP server — show how to use MCP Inspector to manually test tool calls and inspect request/response payloads, then write an integration test that programmatically connects to the server, calls a tool, and asserts on the result. What are the common issues MCP Inspector helps catch that unit tests miss?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. An MCP server works locally over stdio but fails when deployed over HTTP+SSE — walk through the systematic debugging process: diagnosing transport issues (SSE connection drops, CORS, proxy buffering), auth failures (token not forwarded, expired credentials), and timeout problems (long-running tool calls exceeding gateway timeouts). What are the exact steps and tools you use at each stage?</summary>

<!-- Answer will be added later -->

</details>

## Practical — AI Backend Integration

<details>
<summary>24. Implement an API endpoint that streams an LLM response to the client using SSE — show the TypeScript/Node.js code for the endpoint, how you consume the streaming response from the LLM provider's SDK, how you forward chunks to the client, and how you handle errors mid-stream (e.g., the LLM provider drops the connection halfway through). What breaks if you use a regular HTTP response instead of SSE for a slow LLM call?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Build a minimal RAG pipeline — show how to set up pgvector, write an ingestion script that chunks documents, generates embeddings, and stores them, then implement a retrieval API endpoint that takes a query, finds relevant chunks via vector similarity search, and passes them as context to an LLM. What chunking strategy do you use and why, and what goes wrong with naive chunking?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Implement structured data extraction from unstructured text using an LLM — show the TypeScript code that sends a prompt with a schema definition (using tool-use-as-schema or JSON mode), parses the response, validates it against the expected shape (e.g., with Zod), and handles validation failures with a retry or fallback. What happens in production if you skip the validation step?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>27. Implement multi-turn conversation management for a chat feature — show how you store message history (database schema or data structure), how you construct the messages array for the LLM API call, and how you handle hitting the context window limit (truncation strategy or summarization). Include token counting and demonstrate how a token budget prevents runaway costs on long conversations.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. Set up a caching layer for LLM responses to reduce cost and latency — show the implementation for exact-match caching with a normalized cache key strategy, explain TTL considerations and when cached responses go stale. Then explain how semantic caching (matching similar but not identical prompts) works as an extension, what additional infrastructure it requires, and when the complexity of semantic caching is justified vs simple key-based caching.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Add observability to an LLM-powered endpoint — show how you instrument it to log prompt version, token usage, latency, and model, then demonstrate how you trace a multi-step agent flow where one LLM call triggers tool calls that trigger further LLM calls. What makes tracing agent flows harder than tracing traditional request chains?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Set up alerts and a monitoring dashboard for an AI feature — what metrics matter most (token cost, error rate, latency percentiles, output quality proxies), what thresholds trigger alerts, and how do you detect regressions in LLM output quality that don't show up as errors?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you integrated an LLM into a production backend service — what was the feature, what architectural pattern did you choose (sync, async, streaming), what surprised you about LLM behavior in production, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you had to debug a failing AI agent or tool-use chain — what were the symptoms, how did you trace through the multi-step execution to find the root cause, and what guardrails did you add afterward to prevent similar failures?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you had to manage cost or latency for an AI-powered feature — what was the situation, what tradeoffs did you evaluate (model size, caching, batching, async processing), and what measurable impact did your changes have?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you built or consumed an MCP server (or a similar tool integration protocol) — what tools did you expose, what challenges did you face with schema design or transport, and how did the integration change the way the AI system could interact with your services?</summary>

<!-- Answer framework will be added later -->

</details>
