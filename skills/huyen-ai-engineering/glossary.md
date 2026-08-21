# Glossary — AI Engineering (Chip Huyen)

**Agent** — anything that perceives its environment and acts on it, characterized by its environment and tool-augmented action set (Ch 6, p. 488).
**AI judge** — an evaluation system defined by model + prompt + sampling parameters; change any part and you have a different judge (Ch 3, p. 248).
**Autoregressive language model** — predicts the next token from preceding tokens only; the default for text generation (Ch 1, p. 33).
**Benchmark contamination** — a model was trained on its evaluation data, inflating scores; mostly unintentional via web scraping (Ch 4, p. 346).
**Bits-per-byte (BPB)** — cross entropy normalized per byte of original text, comparable across tokenizers (Ch 3, p. 216).
**Catastrophic forgetting** — sequential multi-task finetuning erases earlier tasks; a motivation for model merging (Ch 7, p. 609).
**Chat template** — model-developer-defined format merging system and user prompts; wrong templates degrade quality silently (Ch 5, p. 382).
**Chinchilla scaling law** — compute-optimal training uses ~20× training tokens per model parameter (Ch 2, p. 142).
**Chunking** — splitting documents into retrievable units, with overlap so boundary information survives (Ch 6, p. 474).
**Comparison data** — (prompt, winning, losing) triples used to train reward models (Ch 2, p. 162).
**Compute-bound vs. bandwidth-bound** — workload classification by arithmetic intensity; prefill is compute-bound, decode is memory bandwidth-bound (Ch 9, p. 708-711).
**Context precision / recall** — % of retrieved documents that are relevant / % of relevant documents retrieved (Ch 6, p. 467).
**Contextual retrieval** — augmenting each chunk with metadata or AI-generated context situating it in its document (Ch 6, p. 478).
**Cross entropy** — how hard a model finds predicting a dataset: data entropy plus KL divergence (Ch 3, p. 215).
**Data flywheel** — user-generated data continually improves the product, compounding a first-mover moat (Ch 8, p. 657; Ch 10, p. 817).
**Degenerate feedback loop** — predictions shape feedback which shapes the next model, amplifying bias (Ch 10, p. 845).
**Demonstration data** — labeler-written (prompt, response) pairs for SFT; ~30 min and ~$10 per pair (Ch 2, p. 155-159).
**Distillation** — training a small student to mimic a large teacher via teacher-generated data (Ch 8, p. 686).
**Embedding** — vector capturing meaning; typical size 100-10,000; benchmarked by MTEB (Ch 3, p. 235-237).
**Evaluation-driven development** — defining evaluation criteria before building, mirroring test-driven development (Ch 4, p. 283).
**Exact / semantic caching** — reusing results for identical requests vs. semantically similar ones via embeddings (Ch 10, p. 795-798).
**Factual consistency** — output supported by given context (local) or open knowledge (global); local is far easier (Ch 4, p. 288-289).
**Foundation model** — a general-purpose model built upon for different needs; spans LLMs and LMMs (Ch 1, p. 40).
**Function calling** — model generates tool name + arguments from declared tool specs; valid names are guaranteed, correct values are not (Ch 6, p. 510-513).
**Goodput** — requests per second that satisfy the SLO, versus raw throughput (Ch 9, p. 719).
**Hallucination** — output not grounded in facts; hypothesized causes are self-delusion and labeler-knowledge mismatch (Ch 2, p. 195-198).
**Hybrid search** — combining term-based and embedding-based retrievers, e.g. via reciprocal rank fusion (Ch 6, p. 472-474).
**In-context learning** — teaching desired behavior via examples in the prompt (zero/few-shot), no weight updates (Ch 5, p. 377).
**Instruction hierarchy** — priority order system > user > model outputs > tool outputs; higher wins on conflict (Ch 5, p. 441).
**Inference** — computing output from input; autoregressive decoding is sequential, driving latency (Ch 1, p. 92).
**Jailbreaking** — subverting a model's safety features to make it do bad things (Ch 5, p. 423).
**KV cache** — stores key/value vectors of prior tokens during decoding; grows linearly with sequence length and batch size (Ch 9, p. 749-752).
**LoRA** — low-rank adaptation: train two small matrices A, B and merge W' = W + (α/r)·W_AB; no inference latency (Ch 7, p. 594-596).
**MFU / MBU** — observed throughput vs. peak FLOP/s, and used vs. peak memory bandwidth; the real utilization metrics (Ch 9, p. 720-722).
**Mixture-of-experts (MoE)** — sparse model activating only some parameters per token, e.g. Mixtral 8x7B: 46.7B total, 12.9B active (Ch 2, p. 135).
**Model collapse** — irreversible degradation from recursive training on AI-generated data (Ch 8, p. 684).
**Model gateway** — unified secure interface to all models: access control, cost, fallback, logging (Ch 10, p. 790-793).
**Model router** — intent classifier / next-action predictor sending each query to the optimal solution (Ch 10, p. 788).
**Model merging** — combining models (summing, layer stacking, concatenation) into one multi-task model (Ch 7, p. 608-613).
**Observability** — instrumenting a system so internal state can be inferred from external outputs (Ch 10, p. 804).
**Ossification** — pre-training freezing weights so they adapt poorly during finetuning; hits smaller models more (Ch 8, p. 650).
**pass@k** — fraction of problems where any of k generated samples passes all test cases (Ch 3, p. 226).
**PEFT** — parameter-efficient finetuning: train only small inserted modules or soft prompts (Ch 7, p. 588-592).
**Perplexity** — exponential of cross entropy; the effective number of next-token options (Ch 3, p. 217).
**Post-training** — SFT plus preference finetuning (RLHF/DPO) that turns a completion model into an aligned conversational one (Ch 2, p. 151-152).
**Prompt caching** — reusing overlapping prompt segments (system prompts, docs) so they are processed once (Ch 9, p. 764-766).
**Prompt injection** — malicious instructions injected into prompts, directly or via tool-reachable content (indirect) (Ch 5, p. 424, 429).
**Pruning** — zeroing/removing least-useful parameters; rare in practice since hardware seldom exploits sparsity (Ch 9, p. 738-739).
**QLoRA** — LoRA over a 4-bit NF4-quantized base with paged optimizers; 65B finetunes on one 48 GB GPU (Ch 7, p. 605-606).
**Quantization** — converting values to lower-bit formats to shrink memory and raise throughput (Ch 7, p. 576-577; Ch 9, p. 737).
**Query rewriting** — reformulating ambiguous follow-ups into standalone queries before retrieval (Ch 6, p. 476-477).
**RAG** — retrieval-augmented generation: retrieve relevant external information into the prompt before generating (Ch 6, p. 452).
**ReAct** — interleaving reasoning and action at every agent step: think, act, analyze observation (Ch 6, p. 517).
**Reverse instruction** — using AI to generate the prompts that would elicit existing high-quality human content (Ch 8, p. 677).
**RLHF** — train a reward model on comparison data, then optimize the SFT model with PPO against it (Ch 2, p. 160-165).
**Self-supervision** — the model infers labels from input data itself; what broke the data-labeling bottleneck (Ch 1, p. 36-37).
**Speculative decoding** — a draft model proposes K tokens, the target model verifies them in parallel; no quality change (Ch 9, p. 740-743).
**Task vector** — finetuned minus base weights; add to combine skills, subtract to remove behaviors (Ch 7, p. 614).
**Test time compute** — sampling multiple outputs and picking the best via verifier, logprobs, or majority vote (Ch 2, p. 178-180).
**Token** — basic unit of a language model; for GPT-4 ~3/4 of a word (Ch 1, p. 31-32).
**TTFT / TPOT** — time to first token (prefill) and time per output token (decode); total latency = TTFT + TPOT × output tokens (Ch 9, p. 715-716).
**Violation rate / false refusal rate** — % of attacks that succeed vs. % of safe queries wrongly refused; track both (Ch 5, p. 439).
