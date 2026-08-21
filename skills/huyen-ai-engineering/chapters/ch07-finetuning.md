# Chapter 7: Finetuning

## Core Idea
Finetuning adapts a model by updating its weights instead of its prompt — use it for behavior (form, format, style), use RAG for facts — and because memory is the bottleneck at foundation-model scale, nearly every modern finetuning technique (PEFT, LoRA, QLoRA, quantized training) exists to shrink the memory footprint (p. 544, 559, 563).

## Frameworks Introduced
- **Transfer learning** (p. 544): finetuning is one way to do transfer learning (Bozinovski and Fulgosi, 1976) — knowledge from a data-abundant task (text completion) transfers to a data-scarce task (legal QA, text-to-SQL).
  - When to use: your target task has limited or expensive training data.
  - How: start from a base model whose knowledge overlaps your task; finetuning refines behavior with far fewer samples (sample efficiency — hundreds instead of millions, p. 545).
- **"Finetuning is for form, RAG is for facts"** (p. 559): the decision rule for what to try after prompting is exhausted.
  - When to use: model fails on your task and you must pick the next investment.
  - How: diagnose the failure. Information-based failures (factually wrong, outdated, missing private data) → RAG. Behavioral failures (correct but irrelevant, wrong format, malformed syntax) → finetuning. Both → start with RAG, it's easier (p. 556-559).
- **PEFT — parameter-efficient finetuning** (p. 588): introduced by Houlsby et al. (2019); insert small adapter modules and train only those. Achieved within 0.4% of full finetuning on GLUE with only 3% of trainable parameters (p. 589).
  - When to use: full finetuning doesn't fit your hardware or data budget (it almost never does).
  - How: two buckets — adapter-based (additive) methods that add modules to weights (LoRA, BitFit, IA3, LongLoRA) and soft prompt-based methods that add trainable input tokens (prefix-tuning, P-Tuning, prompt tuning) (p. 590-592).
- **LoRA — Low-Rank Adaptation** (Hu et al., 2021) (p. 594): decompose a weight update into two small matrices A (n×r) and B (r×m); train only A and B; merge back with W' = W + (α/r)·W_AB. Built on **low-rank factorization** (p. 596). No extra inference latency, unlike Houlsby adapters. GPT-3: comparable-or-better performance with ~4.7M trainable parameters — 0.0027% of full finetuning (p. 596).
- **QLoRA — quantized LoRA** (Dettmers et al., 2023) (p. 605): store base weights in 4-bit NF4, dequantize to BF16 for forward/backward pass, plus paged optimizers for CPU-GPU spillover. Lets a 65B model finetune on a single 48 GB GPU (p. 606).
- **Model merging** (p. 608): combine multiple models (often finetunes of the same base) into one. Three approaches: summing (linear combination, SLERP), layer stacking (frankenmerging), concatenation (p. 612-613). Enables multi-task models without catastrophic forgetting and on-device deployment (p. 610).
- **Progression path vs. distillation path** (OpenAI finetuning practices) (p. 625): progression — debug code on the cheapest model, test data on a mid model, push the best model, then map the price/performance frontier. Distillation — finetune the strongest model on a small dataset, use it to generate training data for a cheaper model.

## Key Concepts
- **Continued pre-training**: self-supervised finetuning on cheap task-related data (e.g., raw legal documents) before expensive annotated finetuning (p. 546).
- **Trainable parameter**: a parameter updated during finetuning; each one carries a gradient plus 0-2 optimizer-state values, driving training memory (p. 565, 570).
- **Backpropagation**: forward pass (compute output) + backward pass (loss → gradients → optimizer updates); inference runs only the forward pass (p. 565-566).
- **Quantization**: converting values to a lower-bit format; strictly integer-target only, but used in practice for all precision reduction (p. 576-577).
- **Post-training quantization (PTQ)**: quantize after training — the most common form, and the one relevant to application developers (p. 577).
- **Quantization-aware training (QAT)**: simulate low precision during training so the model performs well when served quantized; does not reduce training cost (p. 582).
- **Gradient checkpointing (activation recomputation)**: don't store activations, recompute them — trades training time for memory (p. 571).
- **Intrinsic dimension**: pre-training implicitly compresses a model's intrinsic dimension; larger, better-trained models have lower intrinsic dimensions, which is why finetuning needs few parameters and samples (p. 596).
- **Task vector (delta parameters)**: finetuned model minus base model; enables task arithmetic — add vectors to combine skills, subtract to remove behaviors (p. 614).
- **Catastrophic forgetting**: sequential multi-task finetuning makes a model forget earlier tasks — a motivation for merging models finetuned in parallel (p. 609-610).
- **Gradient accumulation**: accumulate gradients across several small batches before updating weights, to stabilize training when memory forces small batches (p. 629).

## Mental Models
- Use **RAG when failures are information-based, finetuning when behavioral** — and if both, RAG first with simple term-based retrieval (BM25) before vector databases (p. 559).
- Think of **pre-training as a compression framework**: the better the base model, the fewer parameters and samples finetuning needs (p. 596).
- Use **the number of trainable parameters as your memory dial**: memory for gradients + optimizer states scales with trainable parameters, not total parameters — that is the entire premise of PEFT (p. 564, 570).
- Think of **finetuning vs. RAG as complexity placement**: embedding-based retrieval adds inference-pipeline complexity; finetuning adds development complexity but leaves inference unchanged (p. 561).
- Use **LoRA adapters as swappable task modules**: keep W, A, B separate to serve 100 customers with one base model plus 100 tiny adapters (p. 602-603).

## Anti-patterns
- **Finetuning before systematic prompting**: most "prompting doesn't work" stories collapse under investigation — unclear instructions, unrepresentative examples, undefined metrics (p. 552).
- **Training a domain model because "general models don't do domains"**: BloombergGPT ($1.3M-$2.6M compute, 1.3M A100 hours) was beaten by zero-shot GPT-4-0314 on financial benchmarks the same month it launched (p. 553-554).
- **Loading a model in the wrong numerical format**: Llama 2 shipped in BF16; teams loading it in FP16 got mysteriously degraded quality (p. 576).
- **Single-task finetuning for a multi-task application**: improving "changing orders" queries can degrade product recommendations and feedback — finetune on all query types or use separate/merged models (p. 550-551).
- **Cranking up LoRA rank**: increasing r beyond ~small values often yields no quality gain and may overfit; r between 4 and 64 is usually sufficient (p. 600-601).
- **Merging by concatenation**: no memory savings versus serving models separately; incremental performance rarely worth the parameters (p. 623).

## Code Examples
```text
# Training memory for full finetuning, 13B model, Adam optimizer, 16-bit:
# each trainable parameter = 1 gradient + 2 optimizer states = 3 values
13 billion × 3 × 2 bytes = 78 GB        # gradients + optimizer states alone

# Same model with only 1B trainable parameters (PEFT):
1 billion × 3 × 2 bytes = 6 GB
```
- **What it demonstrates**: why reducing trainable parameters (PEFT) is the highest-leverage memory reduction in finetuning (p. 570).

## Reference Tables

**Numerical formats** (p. 572-574):

| Format | Bits | Bytes | Notes |
|---|---|---|---|
| FP64 (double) | 64 | 8 | NumPy/pandas default; rarely used in NNs |
| FP32 (single) | 32 | 4 | traditional training standard |
| TF32 | 19 (not 32) | — | NVIDIA GPU format |
| BF16 | 16 | 2 | Google/TPU; more range bits, less precision than FP16 |
| FP16 (half) | 16 | 2 | more precision, narrower range (123456.789 → INF) |
| INT8 / INT4 | 8 / 4 | 1 / 0.5 | common inference quantization targets |

**Memory formulas** (p. 568-570):

| Quantity | Formula |
|---|---|
| Weights (inference) | N params × M bytes/param |
| Inference total (approx.) | N × M × 1.2 (activations + KV ≈ 20%) |
| Training total | weights + activations + gradients + optimizer states |
| Gradient + optimizer per trainable param | 1 value (SGD) / 2 (momentum) / 3 (Adam) |

**RAG vs. finetuning on current-events QA** (Ovadia et al., 2024) (p. 558):

| Model | Base | Base + RAG | Base + FT-reg | Base + FT-p |
|---|---|---|---|---|
| Mistral-7B | 0.481 | 0.875 | 0.504 | 0.588 |
| Llama 2-7B | 0.353 | 0.585 | 0.219 | 0.392 |
| Orca 2-7B | 0.456 | 0.876 | 0.511 | 0.566 |

RAG on top of a finetuned model beat RAG alone only 43% of the time on MMLU (p. 560).

**Hyperparameter quick reference** (p. 628-631):

| Knob | Guidance |
|---|---|
| Learning rate | 1e-7 to 1e-3; or pre-training end LR × 0.1-1; fluctuating loss = too big |
| Batch size | <8 can be unstable; use gradient accumulation when memory-bound |
| Epochs | millions of examples: 1-2; thousands: 4-10; val loss up + train loss down = overfit |
| LoRA rank r | 4-64 usually sufficient; α:r ratio typically 1:8 to 8:1 |
| Prompt loss weight | default 10% — learn mostly from responses, a little from prompts |

## Worked Example
Full finetuning a 7B model on consumer hardware — the author's walkthrough (p. 584-585):
1. Weights in FP16: 7B × 2 bytes = **14 GB**.
2. Full finetuning with Adam: each trainable parameter needs 3 extra values (1 gradient + 2 optimizer states) → 7B × 3 × 2 bytes = **42 GB**.
3. Total: 14 + 42 = **56 GB** — before counting activations. Consumer GPUs carry 12-24 GB (48 GB high-end), so full finetuning a mere 7B model is already out of reach.

The escape hatches, in order: cut trainable parameters with LoRA (a Llama 2 13B LoRA adapter at r=2 on query+key is 3.28M params = 6.55 MB against 26 GB of weights, p. 605); cut bits with quantization (QLoRA's 4-bit NF4 base fits 65B on one 48 GB GPU, p. 606); offload to CPU (DeepSpeed, p. 585). Serving side: with 100 customer finetunes on a 4096×4096 matrix (16.8M params), storing 100 merged models costs 1.68B parameters; one shared W plus 100 (A, B) pairs at r=8 costs 23.3M (p. 603).

## Key Takeaways
1. Exhaust systematic prompting first; then diagnose failures — information-based → RAG, behavioral (format, style, syntax) → finetuning; both → RAG first (p. 552, 559, 561).
2. Finetuning memory = weights + activations + gradients + optimizer states; only the last two scale with *trainable* parameters, so PEFT and quantized training attack memory from two independent angles (p. 570, 632).
3. Start with LoRA, not full finetuning: it is parameter- and sample-efficient (good with a few thousand or even hundreds of examples), adds no inference latency once merged, and its modularity enables multi-LoRA serving (p. 604, 626).
4. Apply LoRA to all four attention matrices at low rank rather than fewer matrices at high rank; if you can only pick two, use query and value; feedforward layers can add further gains (p. 599-600).
5. Serve in the lowest precision that holds quality (16/8/4-bit PTQ is standard); train in mixed precision — backpropagation is precision-sensitive (p. 578, 582).
6. Model merging (summing with task-vector pruning à la TIES/DARE, layer stacking for MoE or upscaling) is the multi-task path that avoids catastrophic forgetting, and it can run without GPUs (p. 608, 610, 618-619).
7. "Finetuning is easy, data is hard": frameworks (PEFT, Axolotl, unsloth, LitGPT, LLaMA-Factory) handle training; the real costs are data acquisition, serving, and keeping pace with new base models (p. 551, 627, 632).

## Connects To
- **Ch 1**: finetuning is the costliest rung of ch01's adaptation ladder; BloombergGPT vs GPT-4 is the buy-vs-build lesson made vivid.
- **Ch 2**: supervised and preference finetuning basics come from post-training; numerical formats and model-sizing math carry over (p. 546).
- **Ch 4**: evaluation criteria and pipeline — required before and after any finetuning investment; model selection recurs here (p. 561).
- **Ch 5**: exhaust systematic prompting first (p. 552); few-shot prompt examples convert into finetuning training data.
- **Ch 6**: the decision rule — information-based failures → RAG, behavioral → finetuning; RAG first when both (p. 559).
- **Ch 8**: "finetuning is easy, data is hard" — data quantity depends on PEFT vs full finetuning; distillation runs on synthetic data (p. 547, 650).
- **Ch 9**: quantization reappears at serving time; prompt caching erased finetuning's token-saving advantage; KV cache and memory math shared (p. 555).
- **External**: IEEE 754 floating-point standard (p. 572); federated learning (McMahan et al., 2016) via model merging (p. 610).
