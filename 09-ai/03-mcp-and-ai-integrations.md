# MCP & AI Integrations

> **22 questions** — 14 theory, 5 practical, 3 experience

- MCP core primitives: servers, clients, resources, tools, prompts
- MCP transport layer: stdio vs HTTP+SSE — tradeoffs and security
- MCP tool schemas: JSON Schema definitions and tool call accuracy
- MCP server authentication and authorization patterns
- Building MCP servers in TypeScript with the MCP SDK
- Testing and debugging MCP servers: MCP Inspector, integration tests
- MCP deployment: diagnosing transport, auth, and timeout issues
- Function calling in practice: tool-use loop lifecycle, handling tool call results, error propagation, and timeout management in backend services
- AI agents vs simple API calls: ReAct, plan-and-execute architectures
- Tool use vs RAG: when to give LLMs live access to systems vs pre-indexed knowledge, combining both, failure modes of each
- LLM API backend patterns: streaming responses to clients via SSE/WebSockets, error handling, rate limit management, retry with backoff
- Architectural patterns for AI features: sync, async queues, streaming, background enrichment
- Structured output from LLMs: JSON mode, tool-use-as-schema, validation
- Multi-turn conversation management: message history, context window limits, token budgets
- Integration safety: validating tool calls, preventing infinite agent loops, sandboxing tool execution, prompt injection through tool results
- Observability for LLM calls: logging, tracing multi-step agents, metrics and alerts

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
<summary>9. What backend patterns handle LLM API calls in production — how do you stream responses to clients via SSE or WebSockets instead of waiting for the full response, how do you handle LLM provider errors gracefully (timeouts, 429s, 500s), and what does a robust retry-with-backoff strategy look like for LLM APIs that have unpredictable latency?</summary>

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
<summary>14. How do you add observability to LLM-powered features — what should you log for each LLM call (tokens, latency, model, prompt version), how do you trace a multi-step agent that makes several tool calls, what metrics and alerts matter for catching regressions in cost, latency, or output quality?</summary>

<!-- Answer will be added later -->

</details>

## Practical — MCP Implementation

<details>
<summary>15. Build an MCP server in TypeScript using the MCP SDK that exposes at least two tools and one resource — show the project setup, how you define tool schemas, implement tool handlers, register a resource, and configure the transport. Explain the key decisions in the code and what happens if a tool handler throws an error.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. How do you test and debug an MCP server — show how to use MCP Inspector to manually test tool calls and inspect request/response payloads, then write an integration test that programmatically connects to the server, calls a tool, and asserts on the result. What are the common issues MCP Inspector helps catch that unit tests miss?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. An MCP server works locally over stdio but fails when deployed over HTTP+SSE — walk through the systematic debugging process: diagnosing transport issues (SSE connection drops, CORS, proxy buffering), auth failures (token not forwarded, expired credentials), and timeout problems (long-running tool calls exceeding gateway timeouts). What are the exact steps and tools you use at each stage?</summary>

<!-- Answer will be added later -->

</details>

## Practical — AI Backend Integration

<details>
<summary>18. Implement structured data extraction from unstructured text using an LLM — show the TypeScript code that sends a prompt with a schema definition (using tool-use-as-schema or JSON mode), parses the response, validates it against the expected shape (e.g., with Zod), and handles validation failures with a retry or fallback. What happens in production if you skip the validation step?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Add observability to an LLM-powered endpoint — show how you instrument it to log prompt version, token usage, latency, and model, then demonstrate how you trace a multi-step agent flow where one LLM call triggers tool calls that trigger further LLM calls. What makes tracing agent flows harder than tracing traditional request chains?</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>20. Tell me about a time you integrated an LLM into a production backend service — what was the feature, what architectural pattern did you choose (sync, async, streaming), what surprised you about LLM behavior in production, and what would you do differently?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>21. Describe a time you had to debug a failing AI agent or tool-use chain — what were the symptoms, how did you trace through the multi-step execution to find the root cause, and what guardrails did you add afterward to prevent similar failures?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>22. Tell me about a time you had to manage cost or latency for an AI-powered feature — what was the situation, what tradeoffs did you evaluate (model size, caching, batching, async processing), and what measurable impact did your changes have?</summary>

<!-- Answer framework will be added later -->

</details>