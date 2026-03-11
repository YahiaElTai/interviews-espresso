# LLMs & How They Work

> **34 questions** — 21 theory, 13 practical

- Next-token prediction: how it produces reasoning and its fundamental limitations
- Training vs inference: pretraining, instruction tuning, RLHF alignment — what each stage produces and why it matters for model behavior
- Transformer architecture: self-attention, encoder vs decoder models, why transformers replaced RNNs/LSTMs
- Full inference pipeline: tokenization, embedding, transformer layers, output probabilities, token selection
- Tokenization: how it affects token counts, cost, context limits, and behavior with code/multilingual input
- Embeddings: vector spaces, semantic similarity, distance metrics (cosine, dot product), dimensionality tradeoffs, embedding models
- Context windows: size limits, cost scaling, "lost in the middle" problem, strategies for working within limits
- Sampling parameters: temperature, top-k, top-p
- Model selection: open-source vs proprietary, cost/latency/capability tradeoffs, benchmarks (MMLU, HumanEval), when to use smaller models
- Multi-modal models: vision/image input, audio, practical use cases for backend engineers (document processing, image analysis)
- Hallucinations: causes, mitigation strategies (grounding, retrieval, structured output)
- RAG: chunking, embedding models, vector DBs, retrieval, reranking, diagnosing failure modes
- Prompt engineering vs fine-tuning vs RAG — cost, complexity, accuracy tradeoffs
- Prompting techniques: system prompts, few-shot, chain-of-thought, structured output
- Function calling: how models select and invoke tools, schema design, structured output extraction via tool use
- LLM API characteristics: token-based pricing, rate limits, latency profiles, streaming vs batch, choosing sync vs async calls
- Security risks: prompt injection, data exfiltration, jailbreaking

---

## Foundational

<details>
<summary>1. LLMs are fundamentally next-token predictors — how does predicting the next token one at a time produce what looks like reasoning, planning, and knowledge recall, and what fundamental limitations does this autoregressive approach impose that no amount of scale can fix?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>2. Why did transformers replace RNNs and LSTMs as the dominant architecture for language models -- what specific problems did RNNs have (vanishing gradients, sequential processing bottleneck) that the self-attention mechanism solves, and how does self-attention allow a model to weigh relationships between all tokens simultaneously instead of processing them one at a time?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>3. Walk through the full inference pipeline when you send a prompt to an LLM — from raw text input through tokenization, embedding lookup, transformer layer processing (self-attention + feed-forward), output probability distribution, and finally token selection. How does each stage transform the data, and why does the model need to run this entire pipeline once per generated token?</summary>

<!-- Answer will be added later -->

</details>

## Conceptual Depth

<details>
<summary>4. What is the difference between encoder-only (BERT), decoder-only (GPT), and encoder-decoder (T5) transformer architectures -- why did the field converge on decoder-only models for modern LLMs, what tasks is each architecture best suited for, and why does understanding this distinction matter when choosing between models for different use cases?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>5. How does the three-stage training pipeline -- pretraining on raw text, instruction tuning (supervised fine-tuning on prompt/response pairs), and RLHF alignment -- transform a base model into a useful assistant? What does each stage specifically produce in terms of model behavior, why can't you skip any stage, and how does understanding this pipeline explain common LLM behaviors like sycophancy, refusals, and the gap between base models and chat models?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>6. How does tokenization (specifically BPE — byte-pair encoding) work under the hood, and why does it matter so much for practical LLM usage — how does the tokenizer's vocabulary affect token counts, API costs, context window consumption, and model behavior with code, non-English languages, and unusual formatting?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>7. What are embeddings and why are they central to how LLMs understand language — how do words and tokens get mapped into high-dimensional vector spaces, what geometric properties make these spaces useful (distance metrics like cosine similarity and dot product -- when to use each, clustering, analogy arithmetic), and how do embedding quality and dimensionality affect downstream tasks like search and retrieval?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>8. Why do context windows have hard size limits, how does attention cost scale quadratically with context length, and what architectural innovations (sparse attention, sliding window, RoPE extensions) have been developed to extend context -- what tradeoffs do longer-context models make compared to shorter-context models in terms of cost, latency, and quality?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>9. What is the "lost in the middle" problem -- why do LLMs struggle to use information placed in the center of long contexts even when it's within the window, how has this been demonstrated empirically, and what practical strategies (document ordering, chunking, summarization, iterative refinement) let you work effectively within context limits?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>10. How do the sampling parameters — temperature, top-k, and top-p (nucleus sampling) — shape the probability distribution the model samples from when selecting the next token, why do they exist instead of always picking the highest-probability token, and how do you choose the right settings for different use cases like code generation vs creative writing vs factual Q&A?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>11. Why do LLMs hallucinate — what about the training process and architecture makes hallucination an inherent property rather than a bug to be patched — and what are the most effective mitigation strategies (grounding with retrieved context, structured output constraints, citation requirements, and confidence calibration)?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>12. What is Retrieval-Augmented Generation (RAG) and why does it exist — walk through the full RAG architecture from document ingestion (chunking strategies, chunk size tradeoffs, overlap) through embedding and storage in a vector database, to query-time retrieval and reranking before the LLM generates an answer. Why is each stage necessary?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>13. Why do embedding model selection and reranking matter so much for RAG quality — how do different embedding models (OpenAI, Cohere, open-source like BGE/E5) differ in what they capture, what are the tradeoffs between embedding dimensionality, speed, and retrieval accuracy, and why does a reranker after initial vector search dramatically improve relevance compared to relying on embedding similarity alone?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>14. When should you use prompt engineering vs RAG vs fine-tuning to customize LLM behavior — what are the cost, complexity, accuracy, latency, and maintenance tradeoffs of each approach, and in what scenarios does each one clearly win or clearly fail?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>15. How do you evaluate and select between LLM models for a production use case -- what are the real tradeoffs between proprietary models (GPT-4, Claude) and open-source models (Llama, Mistral), how do you interpret benchmarks like MMLU and HumanEval without being misled by them, and when does using a smaller or cheaper model actually produce better results than defaulting to the most capable option?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>16. What are multi-modal LLMs and why do they matter for backend engineers -- how do vision models process image input alongside text, what practical use cases exist beyond chatbots (document processing, invoice extraction, image analysis for content moderation, diagram-to-code), and what are the cost and latency implications of sending image data compared to text-only requests?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>17. What are the key prompting techniques — system prompts, few-shot examples, chain-of-thought reasoning, and structured output schemas — why does each one improve model output quality, and what are the failure modes when each technique is misapplied or overused?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>18. How does function calling (tool use) work in LLMs — what happens when a model decides to call a function instead of generating text, how does the model select which function to call and with what arguments, and why does this capability fundamentally change what LLMs can do compared to pure text generation?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>19. Why does LLM API integration require different design patterns than traditional API calls — what makes streaming responses necessary for UX, why is retry logic more nuanced with LLMs (idempotency, partial responses, rate limits), and how does the token-based cost model change how you architect features compared to flat-rate APIs?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>20. What are the major security risks when integrating LLMs into applications — how do direct prompt injection, indirect prompt injection (via retrieved content), data exfiltration through crafted outputs, and jailbreaking attacks work, and why are these risks fundamentally harder to defend against than traditional injection attacks like SQL injection?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>21. When a RAG system produces bad answers, how do you systematically diagnose whether the problem is in retrieval (wrong documents found), chunking (relevant info split across chunks), embedding (semantic mismatch between query and content), or generation (model ignoring or misinterpreting retrieved context) — what signals and tests distinguish each failure mode?</summary>

<!-- Answer will be added later -->

</details>

## Practical — Implementation & Integration

<details>
<summary>22. Implement an LLM API integration in TypeScript/Node.js that handles streaming responses -- show how to process tokens as they arrive for real-time UI updates, handle stream interruptions and partial responses, and implement retry logic with exponential backoff for rate limits and transient errors. Explain what breaks in production without proper stream error handling.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>23. Show how to implement token usage tracking and cost controls for an LLM-powered feature in TypeScript -- track token consumption per request and per user, enforce cost budgets that prevent runaway spending, and implement model tiering that routes simpler queries to cheaper models. What monitoring would you add to catch cost anomalies before they become budget problems?</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>24. Build a basic RAG pipeline in TypeScript — show how you chunk documents (with a specific chunking strategy and overlap), generate embeddings, store them in a vector database, retrieve relevant chunks for a user query, and assemble the final prompt with retrieved context. Explain the key decisions at each stage and what happens to answer quality if you get chunk size or overlap wrong.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>25. Show how to implement effective prompting techniques in practice — write a system prompt that constrains model behavior for a specific use case, add few-shot examples that steer output format, use chain-of-thought to improve reasoning accuracy, and enforce structured JSON output with schema validation. Demonstrate what each technique adds and what the output looks like without it.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>26. Implement function calling (tool use) with an LLM API in TypeScript — define tool schemas, handle the model's decision to call a tool, execute the function, return results to the model, and handle the case where the model chains multiple tool calls. Show the code and explain how you prevent the model from calling tools it shouldn't have access to.</summary>

<!-- Answer will be added later -->

</details>

## Practical — Debugging & Hardening

<details>
<summary>27. Your RAG-powered support bot is returning answers that are factually wrong despite the correct documents being in the knowledge base — walk through the exact debugging steps you'd take at each stage (query embedding, retrieval results, chunk inspection, prompt sent to the model, model output) with specific commands or code to test each stage, and show how you identify whether the problem is retrieval, chunking, or generation.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>28. An LLM-powered feature in your application is hallucinating specific details (inventing product names, citing nonexistent policies) in production — walk through how you diagnose the root cause, what combination of grounding, retrieval, output validation, and prompt changes you'd apply to fix it, and how you set up monitoring to catch hallucinations before users see them.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>29. Your team's LLM API costs have tripled in the last month -- walk through a systematic approach to diagnosing where tokens are being spent (which endpoints, which users, prompt vs completion tokens), and show the most impactful cost reduction techniques: prompt optimization to reduce input tokens, semantic caching for repeated or similar queries, and model tiering to route simple tasks to cheaper models. Include TypeScript code for the caching layer and explain the cache invalidation tradeoffs.</summary>

<!-- Answer will be added later -->

</details>

<details>
<summary>30. Your application accepts user input that gets included in LLM prompts — show how you defend against prompt injection attacks at multiple layers: input sanitization, prompt structure that isolates user content, output validation, and privilege separation so the LLM cannot access sensitive tools or data even if injected. Demonstrate a concrete attack and how each defense layer stops it.</summary>

<!-- Answer will be added later -->

</details>

---

## Experience-Based Questions

These questions test real-world experience. Prepare by mapping them to your own projects and situations.

<details>
<summary>31. Tell me about a time you integrated an LLM into a production application — what was the use case, what architecture decisions did you make (prompting strategy, RAG vs fine-tuning, model selection), what unexpected challenges did you face, and how did you measure success?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>32. Describe a time you debugged poor output quality from an LLM-powered feature — what were the symptoms, how did you diagnose whether the issue was in the prompt, the retrieved context, the model choice, or something else, and what changes fixed it?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>33. Tell me about a time you had to choose between prompt engineering, RAG, and fine-tuning for a specific problem — what factors drove the decision, what did you try first, and would you make the same choice again?</summary>

<!-- Answer framework will be added later -->

</details>

<details>
<summary>34. Describe a time you had to address security or trust concerns with an LLM integration — what risks were you worried about (prompt injection, hallucination liability, data leakage), what safeguards did you put in place, and how did you convince stakeholders the system was safe enough to ship?</summary>

<!-- Answer framework will be added later -->

</details>
