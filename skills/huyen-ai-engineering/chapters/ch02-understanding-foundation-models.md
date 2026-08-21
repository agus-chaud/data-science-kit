# Chapter 2: Understanding Foundation Models

## Core Idea
Differences between foundation models trace back to three design decisions — training data, architecture and size, and post-training alignment — plus one runtime lever most people overlook: sampling, which explains hallucinations and inconsistency and can boost performance cheaply (p. 105-106).

## Frameworks Introduced
- **Transformer architecture** (p. 119-129): attention-based architecture that replaced seq2seq/RNNs. Fixes seq2seq's two flaws: decoding from only the final hidden state, and sequential input processing (p. 121). Inference = two phases: **prefill** (input tokens processed in parallel, builds K/V state) and **decode** (one output token at a time) (p. 122-123).
  - When to use this knowledge: latency/cost optimization — prefill parallelizes, decode doesn't; long-context cost comes from storing K/V vectors per previous token (p. 125).
- **Attention mechanism (Q, K, V)** (p. 123-126): query = current decoder state ("person seeking info"), key = page number, value = page content. Attention score = dot product of query and key. Multi-headed: Llama 2-7B splits its 4096-dim vectors across 32 heads of dim 128 (4096/32) (p. 126).
- **Chinchilla scaling law** (p. 142): for compute-optimal training, training tokens ≈ **20× model size**; double the model, double the tokens. A 3B model needs ~60B tokens. Assumes data cost << compute cost; developed for dense models on human data — adaptation for MoE/synthetic data is open research (p. 143).
- **Post-training = SFT + preference finetuning** (p. 151-152): SFT on (prompt, response) demonstration data turns a completion model into a conversational one; preference finetuning (RLHF, DPO, RLAIF) aligns outputs with human preference. Pre-training optimizes token-level quality; post-training optimizes whole-response quality. Shoggoth-with-smiley-face mental image (p. 153-154).
- **RLHF** (p. 160-166): (1) train a reward model on comparison data (prompt, winning_response, losing_response); (2) optimize the SFT model with PPO to maximize reward. Loss: −E_x log(σ(r_θ(x, y_w) − r_θ(x, y_l))) (p. 165). Meta moved Llama 2 (RLHF) → Llama 3 (DPO) to reduce complexity (p. 162).
- **Sampling strategies** (p. 167-177): temperature (divide logits by T before softmax), top-k (softmax over only k logits, k ≈ 50-500), top-p / nucleus (dynamic cutoff at cumulative probability p, typically 0.9-0.95), min-p, stop conditions.
- **Test time compute / best of N** (p. 178-183): sample multiple outputs, pick the best via highest average logprob, a reward model/verifier, majority vote (self-consistency), or app heuristics. OpenAI's verifier gave the same boost as a ~30× model size increase (p. 180).
- **Structured-output stack** (p. 186-191): prompting → post-processing → test time compute → constrained sampling → finetuning. First three are "bandages"; constrained sampling and finetuning are the "intensive treatment".

## Key Concepts
- **Low-resource language**: language with limited training-data availability; English is 45.88% of Common Crawl, 8× Russian (p. 109-110).
- **Sparse model / mixture-of-experts (MoE)**: only a subset of parameters active per token. Mixtral 8x7B: 46.7B total params, 12.9B active — cost/speed of a 12.9B model (p. 135).
- **FLOPs vs FLOP/s**: FLOPs = compute required for a task; FLOP/s = machine throughput. Don't confuse with "FLOPS" (p. 138).
- **Compute-optimal**: best model performance achievable for a fixed FLOP budget (p. 142).
- **Scaling extrapolation (hyperparameter transferring)**: tune hyperparameters on small models, extrapolate to the big one — you get one shot at large scale (p. 145-146).
- **Demonstration data / behavior cloning**: (prompt, response) pairs written by (usually highly educated) labelers for SFT; ~90% of InstructGPT labelers had a college degree; one pair can take 30 min and ~$10 (p. 155-159).
- **Comparison data**: (prompt, winning, losing) triples for reward models; easier to label than absolute scores, still 3-5 min and ~$3.50 per comparison; InstructGPT inter-labeler agreement ~73% (p. 162-163).
- **Logprobs**: log-scale probabilities; avoid underflow, useful for classification, evaluation, debugging — but many providers restrict the logprobs API (p. 174).
- **Inconsistency**: same/similar input → very different outputs (p. 192).
- **Hallucination**: output not grounded in facts; two hypotheses — self-delusion (model can't distinguish its own generations from given data; "snowballing hallucinations") and labeler-knowledge mismatch (SFT teaches the model to answer with knowledge it doesn't have) (p. 195-198).

## Mental Models
- Think of attention as answering questions about a book by referencing any page, versus seq2seq's answering from only the book summary (p. 121-123).
- Think of pre-training as reading to acquire knowledge, post-training as learning how to use it. Post-training is cheap capability-unlocking: InstructGPT spent only 2% of compute on it (p. 152).
- Use three numbers to size up any model: parameters (learning capacity), training tokens (how much it learned), FLOPs (training cost) (p. 140).
- Think of a model's outputs as samples from "the aggregated opinions of the masses" — anything with non-zero probability can come out, which is why the same mechanism powers both creativity and hallucination (p. 191-192).
- Use temperature 0.7 as a starting point for creative tasks; temperature 0 (arg max) for consistency — but consistency is still not guaranteed (hardware, non-fixed seeds) (p. 172, 193).

## Anti-patterns
- **"Use what we have, not what we want" data curation**: models trained on indiscriminate web data (Common Crawl) perform well on tasks in that data, not necessarily yours. 7B tokens of high-quality code let a 1.3B model beat much larger ones (Gunasekar et al., 2023) (p. 108).
- **Translate-to-English pipelines for low-resource languages**: information loss (e.g., Vietnamese relationship pronouns) plus token-cost blowup — Burmese needs ~10× the tokens of English (median 72 vs 7 on MASSIVE), so ~10× latency and API cost (p. 114-115).
- **Judging models by parameter count alone**: misleading for sparse/MoE models; a larger model undertrained on data loses to a smaller well-trained one; Llama 3-8B (2024) beats Llama 2-70B (2023) on MMLU (p. 134-135).
- **Assuming bigger is always better**: inverse-scaling cases exist (memorization tasks, strong-prior tasks); Anthropic found more alignment training can align models *less* (Perez et al., 2022) (p. 141).
- **Trusting JSON mode to solve structure**: it guarantees valid JSON, not correct content; truncation from max-token limits still breaks parsing (p. 185).
- **Scaling test time compute indefinitely**: OpenAI saw gains only up to ~400 samples, then decline (adversarial outputs fool the verifier); nobody in production samples 400+ per query — cost is astronomical (p. 181).
- **Assuming RLHF fixes hallucination**: InstructGPT's RLHF model hallucinated *more* than SFT-only, even though labelers preferred it overall (p. 199).

## Reference Tables
Key numbers to remember (all from this chapter):

| Quantity | Value | Page |
|---|---|---|
| English share of Common Crawl | 45.88% (8× Russian, 5.97%) | p. 110 |
| Burmese vs English median tokens (MASSIVE) | 72 vs 7 (~10× cost/latency) | p. 114 |
| Llama 2-7B: blocks / model dim / FF dim / vocab | 32 / 4,096 / 11,008 / 32K | p. 130 |
| Llama 3-405B: blocks / model dim / FF dim / vocab | 126 / 16,384 / 53,248 / 128K | p. 130 |
| Llama 1 → 2 → 3 training tokens | 1.4T → 2T → 15T | p. 136 |
| Chinchilla: 70B params on 1.4T tokens (vs GPT-3 175B on 300B) | tokens ≈ 20× params | p. 137, 142 |
| GPT-3-175B training compute | 3.14 × 10²³ FLOPs | p. 138 |
| H100 NVL peak | ~60 TeraFLOP/s; 5.2 × 10¹⁸ FLOPs/day | p. 138 |
| GPT-3 training on 256 H100s at peak | ~236 days (~7.8 months); >$4M at 70% util, $2/h | p. 139 |
| Good / great GPU utilization | ~50% / >70% | p. 139 |
| Inference memory floor (7B params, 2 bytes each) | ≥14 GB | p. 134 |
| Mixtral 8x7B total vs active params | 46.7B vs 12.9B (2 of 8 experts/token) | p. 135 |
| Data-center electricity use, now → 2030 | 1-2% → 4-20% global | p. 150 |
| C4 now restricted by ToS/crawling changes | 45% | p. 150 |
| Common temperature / top-k / top-p ranges | 0-2 (0.7 creative) / 50-500 / 0.9-0.95 | p. 172, 175-176 |
| LinkedIn defensive YAML parser | 90% → 99.99% valid outputs | p. 187 |

## Worked Example
**Temperature walk-through (p. 171-172).** Two-token vocabulary {A, B}, logits [1, 2].
- T = 1 (no adjustment): softmax([1, 2]) → [0.27, 0.73]. Model picks B 73% of the time.
- T = 0.5: adjusted logits [1/0.5, 2/0.5] = [2, 4] → softmax → [0.12, 0.88]. B now picked 88% of the time.
- T → 0: probability of B → 1; in practice T = 0 is implemented as arg max (no softmax), since logits can't be divided by zero.
Lower temperature sharpens the distribution toward the highest logit (consistent, boring); higher temperature flattens it, surfacing rare tokens (creative, less coherent).

**Sequence probability for best-of-N (p. 179-180).** Sequence ['I', 'love', 'food'] with token probabilities 0.2, 0.1, 0.3 → sequence probability 0.2 × 0.1 × 0.3 = 0.006. In log scale, logprob(sequence) = sum of token logprobs; divide by length to get average logprob, avoiding bias against long outputs. OpenAI's API picks the sampled output with the highest average logprob.

**GPT-3 training cost (p. 139).** (3.14 × 10²³ FLOPs) / (256 H100s × 5.2 × 10¹⁸ FLOPs/day) ≈ 236 days at peak; at 70% utilization and $2/h per H100, total cost > $4M.

## Key Takeaways
1. Check a model's training-data distribution (language, domain) before adopting it — under-represented languages get worse quality, ~10× worse cost/latency, and different safety behavior (p. 110-115).
2. Size a model by three numbers together — parameters, training tokens, FLOPs — and remember Chinchilla: tokens ≈ 20× parameters for compute-optimal training; production teams (e.g., Llama) deliberately under-size for inference usability (p. 140-144).
3. Post-training is 2% of compute but determines usability: SFT teaches conversation, preference finetuning (RLHF/DPO) teaches values — imperfectly, since "human preference" isn't a single formula (p. 152, 160-162).
4. Tune sampling before tuning models: temperature/top-p for creativity-vs-consistency, best-of-N with a verifier for quality (worth up to a 30× size increase), majority vote for exact-answer tasks (p. 171-182).
5. For structured outputs, escalate: prompting → post-processing → retry → constrained sampling → finetuning; JSON mode alone guarantees syntax, not content (p. 185-190).
6. Treat inconsistency and hallucination as consequences of the probabilistic design, not bugs: cache, fix sampling variables and seed for consistency; for hallucination expect only mitigation (verification, better reward functions, concise-answer prompts) — RLHF can even make it worse (p. 192-199).
7. Scaling has two visible bottlenecks — training data (public data drying up, 45% of C4 now restricted; proprietary data is the moat) and electricity (data centers can grow at most ~50×) (p. 148-150).

## Connects To
- **Ch 1**: instantiates the adaptation framing — training data, architecture, post-training, and sampling are the levers behind ch01's completion machine and last-mile cost curve.
- **Ch 3**: cross entropy/perplexity become evaluation metrics (with the post-training caveat); comparison data feeds preference models and comparative evaluation; sampling parameters are part of an AI judge's identity.
- **Ch 4**: the hallucination hypotheses here motivate factual-consistency measurement (SAFE, SelfCheckGPT); perplexity powers contamination detection; logprob access is a model-selection factor.
- **Ch 5**: a model can't distinguish instructions from data — the root enabler of prompt injection; sampling/robustness and the structured-output stack shape prompting practice.
- **Ch 6**: structured outputs make function calling reliable; context supplied via retrieval mitigates hallucination and inconsistency.
- **Ch 7**: SFT and preference finetuning become techniques you run yourself; Chinchilla-style sizing meets finetuning memory math and numerical formats.
- **Ch 8**: training-data distribution, the drying-up data bottleneck, and AI-generated-data recursion are dataset engineering's starting conditions.
- **Ch 9**: prefill/decode split, attention, and K/V cache are exactly the mechanics inference optimization exploits.
- **External**: Vaswani et al. 2017 (transformer); DeepMind Chinchilla 2022; Mamba/Jamba SSM line (Gu & Dao 2023; Lieber et al. 2024); InstructGPT (Ouyang et al. 2022); DPO (Rafailov et al. 2023).
