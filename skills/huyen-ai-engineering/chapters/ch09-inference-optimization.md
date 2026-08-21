# Chapter 9: Inference Optimization

## Core Idea
Inference optimization makes models faster and cheaper without (ideally) changing their quality; it operates at three levels — model, hardware, and service — and every technique choice starts with identifying whether your workload is compute-bound or memory bandwidth-bound (p. 705-708).

## Frameworks Introduced
- **Compute-bound vs. memory bandwidth-bound** (p. 708-710): from the Roofline paper (Williams et al., 2009). Classify a workload by arithmetic intensity (operations per byte of memory access); profiling tools like NVIDIA Nsight draw the roofline chart.
  - When to use: before picking any optimization or hardware. Compute-bound → more FLOP/s; bandwidth-bound → higher-bandwidth chips.
  - How: prefilling processes input tokens in parallel and is compute-bound; decoding generates one token at a time and is memory bandwidth-bound (p. 710-711). Most LLM inference today is bandwidth-bound (p. 712).
- **Latency decomposition: TTFT + TPOT × output tokens** (p. 715-716): total latency = TTFT + TPOT × (number of output tokens). TTFT corresponds to the prefill step; TPOT to decoding. Use percentiles (p50/p90/p95/p99), never averages — one 3,000 ms outlier makes ten ~100 ms requests average 390 ms (p. 717).
- **Goodput** (p. 719-720): requests per second that satisfy the SLO, adapted from networking. A service completing 100 requests/min where only 30 meet the TTFT/TPOT targets has a goodput of 30 requests/min. Use it instead of raw throughput to avoid optimizing cost into a bad user experience.
- **MFU / MBU** (p. 720-722): MFU (Model FLOP/s Utilization) = observed throughput vs. theoretical peak-FLOP/s throughput. MBU (Model Bandwidth Utilization) = (parameter count × bytes/param × tokens/s) / peak bandwidth. nvidia-smi "GPU utilization" is nearly meaningless — a chip doing 1 of 100 possible ops/s can still report 100% (p. 720).
- **Three-level optimization model (archery analogy)** (p. 736-737): model-level = crafting better arrows (may change model behavior), hardware-level = training a stronger archer, service-level = refining the shooting process (never changes output quality). Production optimization combines levels.
- **Speculative decoding** (p. 740-743): a fast draft model proposes K tokens; the target model verifies them in parallel and accepts the longest agreeing prefix, then generates one extra token. Works because verification is parallelizable (it turns decoding's profile into prefilling's) and decoding leaves idle FLOPs. Easy to implement (~50 lines of PyTorch), no quality change; built into vLLM, TensorRT-LLM, llama.cpp.

## Key Concepts
- **Inference server / inference service**: the server runs models on hardware; the service also receives, routes, and preprocesses requests (p. 706-707).
- **TTFT (time to first token)**: time until the first token after the query; duration of the prefill step; user-visible TTFT can be much longer than model TTFT in CoT/agentic flows — some teams track "time to publish" (p. 714-717).
- **TPOT (time per output token)**: speed of each token after the first. Target ~human reading speed: ~120 ms/token (6-8 tokens/s) suffices for streaming (p. 715).
- **Throughput**: output tokens/s across all users; count input and output throughput separately since prefill and decode are decoupled. Directly linked to cost (p. 718-719).
- **KV cache**: stores key/value vectors of previous tokens so each decoding step computes them only for the newest token; used only at inference, not training. Size = 2 × B × S × L × H × M, grows linearly with sequence length and batch size — 3 TB for a 500B+ model at batch 512/context 2048 (p. 749-752).
- **Quantization**: reducing numerical precision to shrink memory footprint and raise throughput; weight-only quantization is the most popular compression approach — easy, out-of-the-box, extremely effective (p. 737-739).
- **Distillation**: training a small model to mimic a large one; common because the small model can match the large one for your needs (p. 737-739).
- **Pruning**: zeroing or removing least-useful parameters; can cut non-zero params >90% in papers, but rare in practice — hard to do, and hardware often can't exploit sparsity (p. 738-739).
- **Prompt caching** (also context/prefix cache): stores overlapping prompt segments (system prompts, long docs, conversation history) so they're processed once (p. 764-766).
- **Kernel / compiler / lowering**: kernels are hardware-specific optimized routines (CUDA, Triton, ROCm); compilers "lower" model code to hardware, swapping in kernels (e.g., torch.compile, XLA, TensorRT) (p. 755-761).

## Mental Models
- Think of batching as a bus: static batching waits until full; dynamic batching leaves on schedule or when full; continuous (in-flight) batching drops off a passenger and immediately picks up another (p. 761-763).
- Use online APIs when latency matters (chatbots, codegen); use batch APIs (~50% cheaper, hours-scale turnaround) for synthetic data, periodic reports, reindexing, migrations (p. 712-713).
- Think of utilization as a means, not a goal: higher MFU means nothing if cost and latency both increase — you care about jobs done faster and cheaper (p. 724).
- Use the memory hierarchy picture for GPU work: CPU DRAM (25-50 GB/s) → GPU HBM (256 GB/s-1.5+ TB/s) → on-chip SRAM (>10 TB/s, ~40 MB) — most GPU optimization is exploiting this hierarchy (p. 731-733).

## Anti-patterns
- **Optimizing to average latency**: outliers skew it; use p50/p90/p95/p99 and plot TTFT against input length (p. 717).
- **Trusting nvidia-smi GPU utilization**: it measures busy time, not efficiency; use MFU/MBU instead (p. 720-721).
- **Chasing throughput alone**: batching can double or triple throughput at the cost of TTFT/TPOT (LinkedIn, 2024); measure goodput against your SLO (p. 719-720).
- **Comparing providers by tokens/s**: tokenizers differ, so token counts differ; compare cost per request (p. 719).
- **Running prefill and decode on the same machine at scale**: one new prefill job drains compute from all in-flight decode jobs, hurting TPOT; disaggregate them (DistServe) (p. 763-764).
- **Assuming service providers don't change model quality**: model-level optimizations by inference providers cause measurable benchmark variation for the same Llama models (Cerebras, 2024) (p. 736-737).

## Code Examples
```text
MBU = (parameter count × bytes/param × tokens/s) / peak bandwidth

7B model in FP16 at 100 tokens/s:  7B × 2 × 100 = 700 GB/s
On an A100-80GB (2 TB/s):          700 GB/s / 2 TB/s = 70% MBU
```
- **What it demonstrates**: how to compute bandwidth utilization, and why quantization matters — fewer bytes/param consumes less scarce bandwidth (p. 721-722).

## Reference Tables

### Efficiency metrics (p. 714-724)
| Metric | Measures | Notes |
|---|---|---|
| TTFT | Time to first token | Prefill duration; user TTFT ≠ model TTFT with CoT/agents |
| TPOT | Time per token after the first | ~120 ms/token OK for streaming; variants: TBT, ITL |
| Total latency | TTFT + TPOT × output tokens | Report in percentiles |
| Throughput | Output tokens/s (also RPS, RPM) | Directly linked to cost; count prefill/decode separately |
| Goodput | Requests/s meeting the SLO | Guards UX against throughput-only tuning |
| MFU | Observed / peak-FLOP/s throughput | >50% considered good for training; prefill MFU > decode MFU |
| MBU | Used / peak memory bandwidth | High for bandwidth-bound workloads |

### Optimization-technique taxonomy
| Level | Technique | What it does | Quality impact |
|---|---|---|---|
| Model | Quantization (p. 737-739) | Lower precision → smaller, faster | Possible degradation |
| Model | Distillation (p. 737-738) | Small model mimics large | New model behavior |
| Model | Pruning (p. 738-739) | Zero/remove parameters | Rare in practice |
| Model | Speculative decoding (p. 740-743) | Draft model + parallel verification | None |
| Model | Inference with reference (p. 743-744) | Copy draft tokens from input; 2× in retrieval/code/multi-turn | None |
| Model | Parallel decoding: Lookahead/Jacobi, Medusa (p. 746-748) | Generate future tokens simultaneously, then verify | Hard to implement |
| Model | Attention redesign: local windowed, multi-query, grouped-query, cross-layer (p. 752-753) | Shrink KV cache/computation; Character.AI cut KV cache >20× | Train/finetune-time only |
| Model | KV cache management: PagedAttention (vLLM), KV quantization/compression (p. 753) | Reduce fragmentation and cache size | None |
| Model | Kernels: FlashAttention; vectorization, parallelization, loop tiling, operator fusion (p. 754-758) | Hardware-specific compute speedups | None |
| Service | Static/dynamic/continuous batching (p. 761-763) | Raise throughput; continuous returns finished responses immediately | None; latency tradeoff |
| Service | Prefill/decode decoupling (p. 763-764) | Separate machines per phase; ratio 2:1-4:1 for long inputs/TTFT priority, 1:2-1:1 for TPOT priority | None |
| Service | Prompt caching (p. 764-766) | Reuse overlapping segments (system prompts, docs) | None |
| Service | Replica parallelism (p. 767) | More model copies → more concurrent requests, more chips | None |
| Service | Tensor parallelism (p. 768-769) | Split operators across devices; serves big models and cuts latency | None |
| Service | Pipeline parallelism (p. 769-770) | Split model into stages; adds latency — avoid for latency-strict apps | None |
| Service | Context / sequence parallelism (p. 770) | Split long inputs or operators across machines | None |

## Worked Example
**Throughput → cost, then goodput as the decision gate (p. 719-720).** Huyen walks the cost math end-to-end: hardware at $2/h with 100 tokens/s decode throughput costs ~$5.556 per 1M output tokens. At 200 output tokens/request, decoding 1K requests costs $1.11. The same $2/h hardware prefilling 100 requests/min puts prefill for 1K requests at $0.33. Total: $1.44 per 1K requests. The tradeoff: batching could double or triple that throughput (halving cost) — LinkedIn reports this is common — but at the price of worse TTFT and TPOT. The decision tool is goodput: with an SLO of TTFT ≤ 200 ms and TPOT ≤ 100 ms, a service completing 100 requests/min where only 30 meet the SLO has goodput 30 — so the "cheaper" high-throughput configuration may actually deliver less usable capacity. Optimize throughput only up to the point where goodput stops rising.

A second cost decision the author quantifies: prompt caching (p. 765-766). A 1,000-token system prompt at 1M API calls/day means ~1B repeated input tokens daily; Anthropic's cached "chat with a book" (100K-token prompt) drops TTFT from 11.5 s to 2.4 s (-79%) at -90% cost; Gemini discounts cached tokens 75% but charges for cache storage.

## Key Takeaways
1. Diagnose the bottleneck first: prefilling is compute-bound, decoding is memory bandwidth-bound — the right chip and technique differ per phase (p. 710-712).
2. Track TTFT and TPOT separately, in percentiles, against your SLO; use goodput, not raw throughput, when tuning batching (p. 715-720).
3. The highest-impact techniques across use cases: quantization, tensor parallelism, replica parallelism, and attention mechanism optimization (p. 773).
4. Prefer techniques that don't change model quality — speculative decoding, batching, prompt caching, prefill/decode decoupling — before ones that do (quantization, distillation, attention redesign) (p. 736-737, 740-743).
5. Match technique to workload: KV cache work matters for long contexts; prompt caching for overlapping prompts and multi-turn chat; replica parallelism when latency beats cost (p. 772-773).
6. Even if you only consume model APIs, these techniques explain provider pricing (output tokens cost 2-4× input tokens; one output token ≈ 100 input tokens in latency) and quality variation across providers (p. 739, 736).

## Connects To
- **Ch 1**: the sequential-decoding latency problem and usefulness thresholds (latency, cost) named there are what this chapter attacks.
- **Ch 2**: transformer architecture, prefill/decode steps, attention, and the KV cache — the mechanics this chapter optimizes (p. 710, 748).
- **Ch 4**: explains the cost/latency selection criteria (TTFT/TPOT targets, per-token pricing 2-4× output/input) and why the same model varies across providers.
- **Ch 6**: retrieval-heavy and multi-turn workloads are the prime beneficiaries of prompt caching and inference-with-reference; token cost is RAG's ongoing tax.
- **Ch 7**: quantization and numerical representations shared with finetuning; training-vs-inference memory demands; prompt caching erased finetuning's token-saving edge (p. 737, 729).
- **Ch 8**: batch APIs serve synthetic-data generation; model distillation bridges dataset engineering and model compression (p. 737-738).
- **Ch 10**: system-level caches (exact/semantic) complement this chapter's KV/prompt caches; TTFT/TPOT become production monitoring metrics; inference servers sit behind the model gateway (p. 773).
- **Roofline model (Williams et al., 2009)**: the standard HPC framework behind compute-vs-bandwidth analysis (p. 709).
