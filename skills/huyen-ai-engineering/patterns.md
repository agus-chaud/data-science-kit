# Patterns — AI Engineering (Chip Huyen)

## Crawl-Walk-Run Automation Ramp
**Source**: Ch 1, p. 73
**When to use**: rolling out an AI feature whose reliability is unproven.
**How**: Crawl — human involvement mandatory; Walk — AI interacts with internal employees; Run — direct AI interaction with external users. Promote a slice when acceptance is high (e.g., 95% of AI drafts used verbatim → let AI answer those directly).
**Trade-offs**: slower rollout, but each stage generates the acceptance data justifying the next.

## Best-of-N / Test Time Compute
**Source**: Ch 2, p. 178-183
**When to use**: quality matters more than cost/latency and you can score candidates.
**How**: sample N outputs; pick the winner via highest average logprob, a reward model/verifier, majority vote (exact-answer tasks), or app heuristics.
**Trade-offs**: a verifier gave gains equal to ~30× model size, but returns decline past ~400 samples and cost scales linearly with N.

## Structured-Output Escalation Stack
**Source**: Ch 2, p. 186-191
**When to use**: outputs must parse (JSON, YAML, SQL).
**How**: escalate prompting → post-processing → retry/test-time compute → constrained sampling → finetuning. LinkedIn's defensive YAML parser took 90% → 99.99% valid.
**Trade-offs**: first three are cheap "bandages"; constrained sampling and finetuning cost engineering but actually guarantee structure. JSON mode guarantees syntax, not content.

## AI-as-Judge Prompt Design
**Source**: Ch 3, p. 239-249
**When to use**: open-ended criteria (relevance, faithfulness) with no reference data; production monitoring.
**How**: prompt = task + detailed criteria + scoring system; prefer classification or discrete 1-5 over continuous scales; include scored examples with justifications; pin and version model + prompt + sampling.
**Trade-offs**: examples raised GPT-4 judge consistency 65% → 77.5% but 4× cost; judges carry self-, position-, and verbosity biases; scores are not comparable across judges.

## Comparative Evaluation (Pairwise Ranking)
**Source**: Ch 3, p. 262-267
**When to use**: subjective quality where "which is better" is easier than "how good".
**How**: show evaluators two responses, record winners, convert win rates to a ranking via Elo/Bradley-Terry.
**Trade-offs**: a win rate cannot tell you whether either model is good enough or worth its cost — pair with absolute metrics; only use competent voters.

## Three-Step Evaluation Pipeline
**Source**: Ch 4, p. 351-366
**When to use**: any application before and after launch.
**How**: (1) evaluate every component, per turn and per task; (2) write a guideline defining *good* (not just correct), 2-3 criteria, rubrics with scored examples, tied to business metrics; (3) mix methods (cheap classifier on 100% traffic, AI judge on 1%), slice data, bootstrap the eval set, evaluate the pipeline itself.
**Trade-offs**: guideline work is slow but amortizes — it doubles as annotation guidelines for finetuning data (Ch 8, p. 660).

## Prompt Decomposition
**Source**: Ch 5, p. 399-404
**When to use**: one mega-prompt handles multiple concerns poorly.
**How**: split into chained subtask prompts (e.g., intent classification → per-intent response); use cheaper models for easy steps. GoDaddy decomposed a 1,500-token prompt: better performance, lower cost.
**Trade-offs**: more API calls and perceived latency, but monitoring, debugging, and parallelization improve.

## Chain-of-Thought + Self-Critique
**Source**: Ch 5, p. 405-409
**When to use**: reasoning tasks; hallucination reduction (LinkedIn's finding).
**How**: "think step by step", specify steps, or show a worked one-shot example; then ask the model to check its own output.
**Trade-offs**: cheap, works across models; costs latency because intermediate tokens precede the answer.

## Defense-in-Depth Against Prompt Attacks
**Source**: Ch 5, p. 439-445
**When to use**: any app exposed to user input or untrusted tool content.
**How**: model level — instruction-hierarchy finetuning (up to +63% robustness); prompt level — explicit prohibitions, repeat system prompt after untrusted content; system level — isolate execution, human approval for impactful actions, guardrail inputs and outputs.
**Trade-offs**: repeated system prompt doubles its tokens; measure false refusal rate alongside violation rate or you ship a useless over-refuser.

## Hybrid Search with Reciprocal Rank Fusion
**Source**: Ch 6, p. 472-474
**When to use**: queries mixing exact strings (error codes, product names) with semantic intent.
**How**: run term-based (BM25) and embedding-based retrievers in parallel; fuse with Score(D) = Σ 1/(k + rᵢ(D)), k ≈ 60 — or sequentially: cheap retriever fetches, expensive one reranks.
**Trade-offs**: embedding infra can cost 1/5 to 1/2 of model API spend; term-based alone misses semantics, embeddings alone miss keywords.

## Plan-Validate-Execute-Reflect Agent Loop
**Source**: Ch 6, p. 500-520
**When to use**: multi-step tasks with tools.
**How**: generate plan → validate it (heuristics or AI judge) before running → execute via function calling → reflect on outcomes (ReAct interleaves; Reflexion splits evaluator from self-reflection) → loop.
**Trade-offs**: reflection isn't required to operate but is necessary to succeed; costs tokens/latency, and agents can falsely conclude success — verify externally.

## LoRA / QLoRA Finetuning
**Source**: Ch 7, p. 594-606
**When to use**: default finetuning approach — full finetuning of even 7B needs ~56 GB.
**How**: train low-rank A, B (r = 4-64) on all four attention matrices (query+value if only two); merge for zero-latency serving, or keep adapters separate to serve many customers off one base. QLoRA: 4-bit NF4 base, BF16 compute.
**Trade-offs**: slightly below full finetuning in some cases; raising r beyond small values rarely helps and can overfit; unmerged adapters add serving logic.

## Small-Dataset Probe + Scaling Curve
**Source**: Ch 8, p. 652-655
**When to use**: before committing annotation budget.
**How**: finetune on ~50 crafted examples; improvement predicts more data will pay. Then finetune on 25/50/100% subsets and plot the curve — steep slope means double the data, plateau means stop.
**Trade-offs**: a null probe can also mean bad hyperparameters or prompts — rule those out before concluding.

## Synthetic Data with Paired Verification
**Source**: Ch 8, p. 678-688
**When to use**: scaling instruction data beyond what you can annotate.
**How**: generate problems → generate solutions (CoT in prompt) → verify statically (linters) → verify dynamically (unit tests) → self-correct on failures (~20% recovered) → multiply via translation and back-translation, filtering failures. Llama 3: 2.7M verified coding examples.
**Trade-offs**: only verified data enables self-improvement; unverified self-generated data degrades models and risks model collapse — mix in real data.

## Speculative Decoding
**Source**: Ch 9, p. 740-743
**When to use**: cutting decode latency with zero quality change.
**How**: fast draft model proposes K tokens; target model verifies in parallel, accepts longest agreeing prefix, generates one more. ~50 lines of PyTorch; built into vLLM, TensorRT-LLM, llama.cpp.
**Trade-offs**: needs a well-matched draft model; wasted draft compute when acceptance is low.

## Prompt Caching
**Source**: Ch 9, p. 764-766
**When to use**: overlapping prompt segments — long system prompts, docs, multi-turn chat.
**How**: cache the shared prefix so it's processed once. Anthropic's 100K-token cached prompt: TTFT 11.5 s → 2.4 s (-79%), -90% cost.
**Trade-offs**: providers may charge cache storage (Gemini: 75% token discount but storage fees); cache design constrains prompt ordering (shared prefix first).

## Router + Gateway Architecture
**Source**: Ch 10, p. 788-794
**When to use**: multiple models/providers, cost pressure, out-of-scope traffic.
**How**: intent-classifier router sends simple queries to cheap models, out-of-scope ones to stock responses without an API call; a model gateway unifies access, cost tracking, fallback, and logging across providers.
**Trade-offs**: routers must be fast and cheap because they stack; routed models have different context limits.

## Implicit Feedback Design
**Source**: Ch 10, p. 836-839
**When to use**: harvesting training signal without asking users to rate anything.
**How**: make normal workflow actions carry signal — Midjourney's upscale (strong positive) / vary (weak) / regenerate (negative); Copilot's Tab-to-accept. Treat user edits as preference pairs (original = losing, edited = winning).
**Trade-offs**: signals are noisy (curiosity regenerations); standalone chat apps outside the workflow collect fundamentally weaker feedback than integrated products.
