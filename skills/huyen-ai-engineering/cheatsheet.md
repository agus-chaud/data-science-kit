# Cheatsheet — AI Engineering (Chip Huyen)

## Adaptation decision tree
- Model fails your task →
  - Prompting not yet systematic (no versioning, no eval metrics)? → fix prompting first; most "prompting doesn't work" collapses under investigation (Ch 5, p. 374; Ch 7, p. 552)
  - Failure is **information-based** (wrong/outdated/missing private facts) → RAG (Ch 7, p. 556-559)
  - Failure is **behavioral** (format, style, syntax) → finetune — "finetuning is for form, RAG is for facts" (Ch 7, p. 559)
  - Both → RAG first, it's easier; start with BM25 before vector DBs (Ch 7, p. 559)
- Knowledge base < ~200K tokens (~500 pages) → skip RAG, put it all in the prompt (Ch 6, p. 454)
- Never train a domain model because "general models can't": BloombergGPT ($1.3-2.6M) lost to zero-shot GPT-4 the month it shipped (Ch 7, p. 553-554)

## Host vs. API
- Data can't leave org / need logprobs / need to freeze the model → self-host (Ch 4, p. 325-332)
- Need best quality, scaling, function calling out of the box → API; strongest models stay closed (Ch 4, p. 327-329)
- Cost: API is flat per token; self-host cost drops with volume — revisit at each scale change (Ch 4, p. 314)

## Evaluation
- Method fallback chain: functional correctness → similarity vs. references → AI judge (Ch 3, p. 224-240)
- Public benchmarks/leaderboards: only to filter OUT models, never to pick the winner — assume contamination (Ch 4, p. 346, 350)
- Judge design: discrete 1-5 or classification, never wide/continuous scales; pin model+prompt+sampling; judge at temperature 0 (Ch 3, p. 245-248; Ch 4, p. 365)
- Eval-set size: detect 30% diff → ~10 samples; 10% → ~100; 3% → ~1,000; 1% → ~10,000 (Ch 4, p. 364)
- 2-3 criteria per app (users average 2.3); layer cheap classifier on 100% of traffic, AI judge on ~1% (Ch 4, p. 355, 359)
- Slice data or risk Simpson's paradox: A can win every subset and lose overall (Ch 4, p. 361-362)
- Don't use perplexity on post-trained or quantized models (Ch 3, p. 221)

## Sampling & outputs
- Temperature 0.7 creative / 0 consistency; top-p 0.9-0.95; top-k 50-500 (Ch 2, p. 172-176)
- Best-of-N: verifier worth ~30× model size; gains stop past ~400 samples (Ch 2, p. 180-181)
- Structure: prompt → post-process → retry → constrained sampling → finetune; JSON mode ≠ correct content (Ch 2, p. 185-190)
- Critical instructions at prompt beginning or end, never the middle (Ch 5, p. 386-387)

## Finetuning defaults
- Start with LoRA: r 4-64, all four attention matrices (q+v if only two), α:r 1:8-8:1 (Ch 7, p. 599-601, 628)
- LR 1e-7 to 1e-3 (or pre-train end LR × 0.1-1); fluctuating loss → LR too big (Ch 7, p. 628)
- Epochs: millions of examples 1-2; thousands 4-10; val loss up + train loss down → overfit (Ch 7, p. 629)
- Batch < 8 unstable → gradient accumulation (Ch 7, p. 629)
- Memory: trainable params × 3 values × 2 bytes (Adam, 16-bit); inference ≈ weights × 1.2 (Ch 7, p. 568-570)
- Load in the shipped format: Llama 2 BF16 loaded as FP16 silently degrades (Ch 7, p. 576)
- Chinchilla (pre-training only): tokens ≈ 20× params (Ch 2, p. 142)

## Data
- Probe with ~50 crafted examples before scaling spend; then 25/50/100% curve — plateau means stop buying data (Ch 8, p. 652-655)
- Small data (hundreds-thousands) → PEFT on an advanced model; large data → full finetuning on a smaller model (Ch 8, p. 651-652)
- Quality beats quantity: 1K curated (LIMA) rivaled GPT-4 in 43% of cases; 0.1% duplicated 100× halved effective model size (Ch 8, p. 643, 693)
- Synthesize only what you can verify; never train recursively on unverified self-generated data (Ch 8, p. 681-688)
- Inference prompt format must match training format exactly (Ch 8, p. 699)

## Inference optimization
- Diagnose first: prefill = compute-bound, decode = bandwidth-bound (most LLM inference) — pick chip/technique per phase (Ch 9, p. 710-712)
- Prefer quality-neutral wins first: speculative decoding, batching, prompt caching, prefill/decode decoupling — then quantization/distillation (Ch 9, p. 736-743)
- Highest-impact overall: quantization, tensor parallelism, replica parallelism, attention optimization (Ch 9, p. 773)
- Targets: TPOT ~120 ms/token suffices for streaming; report p50/p90/p99, never averages; tune batching by goodput, not throughput (Ch 9, p. 715-720)
- Prefill:decode machine ratio — 2:1-4:1 for long inputs/TTFT priority; 1:2-1:1 for TPOT priority (Ch 9, p. 763-764)
- Latency-tolerant work (synthetic data, reindexing) → batch API, ~50% cheaper (Ch 9, p. 712-713)
- Don't trust nvidia-smi utilization — use MFU/MBU (Ch 9, p. 720-721)

## Agents & architecture
- Agent error compounds: 95%/step → 60% over 10 steps → use stronger models than single-shot needs (Ch 6, p. 491)
- Validate plans before executing; human approval on write actions; ablate tools — remove any whose absence costs nothing (Ch 6, p. 500, 522-523)
- Build order: bare API → context → guardrails → router/gateway → caches → agent patterns; add only on concrete need (Ch 10, p. 779-800)
- Don't cache user-specific/time-sensitive queries (leak risk); don't default to semantic caching (Ch 10, p. 796-798)
- Mask PII before it leaves the org; delay orchestrators until the system works without one (Ch 10, p. 782, 815)

## Tells & smells
- Low perplexity on a benchmark → it leaked into training data (Ch 3, p. 222)
- Model answers "reasonably" but worse than expected → wrong chat template; print the final prompt (Ch 5, p. 383-384)
- Judge scores drift month-over-month → unversioned judge, not your app (Ch 3, p. 253-254)
- Zero violation rate → check false refusal rate; you may have built a refuser (Ch 5, p. 439)
- Sudden production regression with no deploy → silent provider model swap (Voiceflow lost 10%) (Ch 10, p. 811-812)
- Users answering "Are you sure?" → distrust signal; user edits → free preference pairs (Ch 10, p. 820-822)
