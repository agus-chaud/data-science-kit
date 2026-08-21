---
name: huyen-ai-engineering
description: "Knowledge base from \"AI Engineering: Building Applications with Foundation Models\" by Chip Huyen (O'Reilly). Use when applying Huyen's frameworks for foundation models, evaluation, prompt engineering, RAG, agents, finetuning, dataset engineering, inference optimization, or AI application architecture — or when studying the book or referencing its concepts."
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# AI Engineering: Building Applications with Foundation Models
**Author**: Chip Huyen | **Pages**: ~905 | **Chapters**: 10 | **Generated**: 2026-07-20

## How to Use This Skill

- **Without arguments** — load core frameworks for reference
- **With a topic** — ask about `RAG`, `quantization`, or another indexed topic; I find and read the relevant chapter
- **With chapter** — ask for `ch05`; I load that specific chapter
- **Browse** — ask "what chapters do you have?" to see the full index

When you ask about a topic not covered in Core Frameworks below, I will read
the relevant chapter file before answering.

---

## Core Frameworks & Mental Models

### Adaptation ladder: prompting → RAG → finetuning (p. 43, 87)
Three adaptation techniques in increasing order of data and complexity. Exhaust each rung before the next. Most "prompting doesn't work" collapses under investigation — prompt systematically first (versioned prompts, eval metrics, whole-system evaluation) before touching heavier machinery (p. 374, 552).

### "Finetuning is for form, RAG is for facts" (p. 559)
The decision rule once prompting is exhausted. Diagnose the failure:
- **Information-based** (wrong, outdated, or missing private facts) → RAG
- **Behavioral** (correct info, wrong format/style/syntax) → finetuning
- **Both** → RAG first, it's easier; start with BM25 before vector DBs
- Knowledge base under ~200K tokens (~500 pages) → skip RAG, put it all in the prompt (p. 454)
- Never train a domain model because "general models can't": BloombergGPT ($1.3–2.6M) lost to zero-shot GPT-4 the month it shipped (p. 553-554)

### Host vs. API — build-vs-buy along seven axes (p. 314, 324-333)
Privacy, lineage, performance, functionality, cost, control, on-device. Self-host when data can't leave the org, you need logprobs, or you must freeze the model. Use APIs for best quality and scaling out of the box — the strongest models stay closed. API cost is flat per token; self-host cost drops with volume — revisit at each scale change.

### Evaluation-driven development (p. 283)
Define evaluation criteria BEFORE building, inspired by TDD. An app you can't evaluate is worse than one never deployed. Criteria in four buckets: domain-specific capability, generation capability, instruction-following, cost/latency. Evaluate each with the cheapest trustworthy method.

### Four-step model selection (p. 318)
(1) Filter by hard attributes (license, privacy, size — things you can't change); (2) shortlist via public benchmarks; (3) run YOUR evaluation pipeline to pick the winner; (4) monitor and iterate. Public benchmarks only filter OUT models, never pick the winner — assume contamination (p. 346, 350).

### Evaluation method fallback chain (p. 224-240)
Prefer functional correctness (pass@k, execution accuracy) → similarity vs. references (exact, lexical, semantic) → AI as a judge. A judge = model + prompt + sampling parameters: pin all three, use discrete scales (1-5 or classification, never wide/continuous), judge at temperature 0, never compare scores across differently configured judges (p. 245-248, 253). Size eval sets by detectable difference: 10% diff ≈ 100 samples, 1% ≈ 10,000 (p. 364). Don't use perplexity on post-trained or quantized models (p. 221).

### Prefill/decode asymmetry — the root of inference optimization (p. 122, 710-712)
Prefill processes input tokens in parallel (compute-bound); decode generates one token at a time (memory bandwidth-bound — most LLM inference today). Diagnose the bottleneck before picking any technique or chip. Latency = TTFT + TPOT × output tokens; report percentiles, never averages (p. 715-717).

### Goodput over throughput (p. 719-720)
Requests/sec that MEET the SLO, not raw requests/sec. Tune batching only until SLO-meeting throughput stops rising — raw throughput optimization can cost you the user experience. Prefer quality-neutral wins first: speculative decoding, batching, prompt caching, prefill/decode decoupling — then quantization and distillation (p. 736-743).

### Three layers of the AI stack (p. 81-82)
Application development (prompts, context, evaluation, interfaces — most post-2022 action), model development (training, dataset engineering, inference optimization), infrastructure (serving, monitoring — least changed). AI engineering vs. ML engineering: less modeling, more adaptation; more inference optimization; evaluation becomes the hardest problem because outputs are open-ended (p. 86).

### Crawl-Walk-Run automation levels (p. 73)
Crawl: human involvement mandatory. Walk: AI interacts with internal employees. Run: direct AI interaction with external users. Promote as measured acceptance rises. Pairs with agent safety: human approval on write actions — "you shouldn't give an intern authority to delete your production database" (p. 497).

### Agent task-solving loop (p. 503)
Plan generation → plan reflection/validation → execution (function calling) → outcome reflection; loop. Decouple planning from execution; validate plans before running (p. 500). Agent errors compound: 95% per-step accuracy → 60% over 10 steps — agents need stronger models than single-shot tasks (p. 491). Ablate tools: remove any whose absence costs nothing (p. 522-523).

### Instruction hierarchy + three-level defense (p. 439-445)
Priority: system prompt > user prompt > model outputs > tool outputs — tool outputs lowest neutralizes many indirect injections. Defend at model level (hierarchy training), prompt level (explicit prohibitions, repeated system prompt), system level (isolation, human approval, guardrails). Measure violation rate AND false refusal rate — zero violations may mean you built a refuser (p. 439).

### Data curation: quality / coverage / quantity (p. 642)
Probe with ~50 crafted examples before scaling spend; then plot 25/50/100% subset curves — plateau means more data won't pay (p. 652-655). Quality beats quantity: 1K curated examples (LIMA) rivaled GPT-4 in 43% of cases (p. 643). Synthesize only what you can verify; never train recursively on unverified self-generated data — model collapse (p. 681-688).

### Data flywheel (p. 74, 657, 817)
A startup's moat is first-to-market + usage data. Application data exactly matches your target distribution. Close the loop with feedback design: user edits are free preference pairs; "Are you sure?" answers signal distrust (p. 820-822).

### Progressive architecture build-up (p. 779-800)
Start query → model API → response. Add in order of concrete need: (1) context/retrieval ("feature engineering for foundation models"), (2) input/output guardrails, (3) router + gateway, (4) exact/semantic caches, (5) agent patterns with write actions. Each addition brings capability AND new failure modes. Delay orchestrators until the system works without one. Watch silent drift: undisclosed provider model swaps cost Voiceflow 10% (p. 811-812).

### LoRA defaults (p. 594-604, 628)
Start with LoRA, not full finetuning: r 4-64 on all four attention matrices (q+v if only two), α:r between 1:8 and 8:1, no inference latency once merged. QLoRA = 4-bit base + LoRA: 65B finetune on one 48 GB GPU (p. 605-606). Small data → PEFT on an advanced model; large data → full finetuning on a smaller model (p. 651-652).

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-intro-foundation-models.md) | Introduction to Building AI Applications with Foundation Models | three adaptation techniques, Crawl-Walk-Run, three-layer AI stack |
| [ch02](chapters/ch02-understanding-foundation-models.md) | Understanding Foundation Models | transformer prefill/decode, Chinchilla scaling, post-training (SFT/RLHF), sampling |
| [ch03](chapters/ch03-evaluation-methodology.md) | Evaluation Methodology | perplexity family, functional correctness, AI as a judge, comparative evaluation |
| [ch04](chapters/ch04-evaluate-ai-systems.md) | Evaluate AI Systems | evaluation-driven development, four-step model selection, hard vs. soft attributes |
| [ch05](chapters/ch05-prompt-engineering.md) | Prompt Engineering | prompt anatomy, in-context learning, prompt-attack taxonomy, instruction hierarchy |
| [ch06](chapters/ch06-rag-and-agents.md) | RAG and Agents | RAG pipeline, hybrid search/RRF, agent task-solving loop, ReAct/Reflexion |
| [ch07](chapters/ch07-finetuning.md) | Finetuning | form-vs-facts rule, PEFT/LoRA/QLoRA, model merging, progression vs. distillation |
| [ch08](chapters/ch08-dataset-engineering.md) | Dataset Engineering | quality/coverage/quantity, small-dataset probe, data flywheel, reverse instruction |
| [ch09](chapters/ch09-inference-optimization.md) | Inference Optimization | compute- vs. bandwidth-bound, TTFT/TPOT, goodput, speculative decoding |
| [ch10](chapters/ch10-architecture-user-feedback.md) | AI Engineering Architecture and User Feedback | progressive architecture build-up, guardrails, conversational feedback taxonomy |

## Topic Index

- **Agents (planning, tools, reflection)** → ch06
- **AI as a judge** → ch03, ch04
- **Batching & goodput** → ch09
- **Benchmarks & leaderboards (contamination)** → ch04
- **Caching (prompt, exact, semantic)** → ch09, ch10
- **Chain-of-thought** → ch05
- **Chat templates** → ch05, ch08
- **Chinchilla scaling law** → ch02
- **Context construction** → ch05, ch06, ch10
- **Data flywheel** → ch01, ch08, ch10
- **Distillation** → ch07, ch09
- **Embeddings & vector search** → ch03, ch06
- **Evaluation pipelines** → ch03, ch04
- **Feedback design** → ch10
- **Finetuning (when & how)** → ch07
- **Function calling / structured outputs** → ch02, ch06
- **Guardrails & prompt attacks** → ch05, ch10
- **Hallucination & factual consistency** → ch02, ch04
- **Hosting vs. API (build-vs-buy)** → ch04
- **In-context learning (zero/few-shot)** → ch05
- **KV cache** → ch02, ch09
- **Latency (TTFT, TPOT, percentiles)** → ch09, ch10
- **LoRA / QLoRA / PEFT** → ch07
- **Model merging** → ch07
- **Model selection** → ch04
- **Perplexity & logprobs** → ch02, ch03
- **Post-training (SFT, RLHF, DPO)** → ch02, ch07
- **Prompt engineering** → ch05
- **Quantization** → ch07, ch09
- **RAG (retrieval, chunking, reranking)** → ch06
- **Routers & gateways** → ch10
- **Sampling (temperature, top-p, best-of-N)** → ch02
- **Speculative decoding** → ch09
- **Synthetic data & model collapse** → ch08

## Supporting Files

- [glossary.md](glossary.md) — all key terms with definitions
- [patterns.md](patterns.md) — all techniques and design patterns
- [cheatsheet.md](cheatsheet.md) — quick reference tables and decision guides
- [concept-map.md](concept-map.md) — how concepts link across chapters, prerequisite chains

---

## Scope & Limits

This skill covers the book content only. For hands-on implementation in your codebase,
combine with project-specific tools. For topics beyond this book, check related skills
or ask the agent directly.
