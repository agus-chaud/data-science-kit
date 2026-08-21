# Concept Map — AI Engineering (Chip Huyen)

## Concept → Chapters
- **Adaptation ladder (prompting → RAG → finetuning)** → ch01 (introduced, p. 43, 87), ch05 (rung 1: prompting), ch06 (rung 2: context/RAG/agents), ch07 (rung 3 + decision rule "finetuning is for form, RAG is for facts", p. 559)
- **Evaluation of open-ended outputs** → ch01 (named the discipline's hardest problem, p. 86, 94), ch03 (methodology: perplexity, exact eval, AI judge, comparative), ch04 (evaluation-driven development, pipelines, model selection), ch08 (eval guidelines reused as annotation guidelines, p. 660), ch10 (production monitoring, drift)
- **AI as a judge** → ch03 (introduced, p. 239), ch04 (pipeline component, factual consistency), ch06 (plan validation, reflection evaluators), ch08 (synthetic-data verification), ch10 (monitoring metrics)
- **Perplexity / logprobs** → ch02 (logprobs, sampling), ch03 (introduced, p. 212-223), ch04 (contamination detection, p. 348), ch08 (dedup, abnormality filtering)
- **Post-training & preference data (SFT/RLHF/DPO)** → ch02 (introduced, p. 151-166), ch03 (comparison data → preference models, comparative eval), ch07 (finetuning mechanics, PEFT), ch08 (instruction/preference datasets), ch10 (user edits as preference pairs, p. 822)
- **Hallucination / factual consistency** → ch02 (causes, p. 195-199), ch04 (measurement: SAFE, SelfCheckGPT, NLI), ch05 (CoT + context mitigation), ch06 (RAG grounding), ch10 (output guardrails)
- **Embeddings & vector search** → ch03 (introduced, MTEB, p. 235-237), ch06 (semantic retrieval, ANN algorithms), ch08 (semantic dedup), ch10 (semantic caching)
- **Context construction** → ch01 (RAG named, p. 43), ch05 (context part of prompt, p. 397), ch06 (core topic), ch10 (architecture step 1, "feature engineering for foundation models")
- **Prompt attacks & guardrails** → ch02 (root cause: model can't tell input from instruction), ch05 (attack taxonomy + 3-level defense, p. 419-445), ch06 (write-action risk), ch10 (input/output guardrail components)
- **Transformer prefill/decode & KV cache** → ch02 (introduced, p. 119-129), ch07 (memory math), ch09 (main optimization target, p. 710, 749)
- **Quantization** → ch07 (formats, PTQ/QAT, QLoRA, p. 572-582, 605), ch09 (serving-side compression, p. 737)
- **Distillation & synthetic data** → ch02 (data bottleneck, AI-data recursion), ch07 (distillation path, p. 625), ch08 (synthesis + verification, model collapse), ch09 (model compression)
- **Latency/cost metrics (TTFT, TPOT, goodput)** → ch01 (usefulness threshold, p. 76), ch04 (selection criteria table, p. 315), ch09 (full decomposition, p. 715-722), ch10 (production monitoring, p. 807)
- **Data flywheel** → ch01 (startup moat, p. 74), ch08 (introduced, p. 657), ch10 (feedback design closes it, p. 817)
- **Scaling laws & sizing** → ch02 (Chinchilla, p. 142), ch08 (data-quantity scaling curves, p. 654)
- **Structured outputs / function calling** → ch02 (escalation stack, p. 186), ch06 (function calling for agents, p. 510)
- **Instruction-following capability** → ch04 (IFEval/INFOBench, p. 301), ch05 (prerequisite for prompting)
- **Chat templates** → ch05 (introduced, p. 382), ch08 (dictate finetuning data format, p. 697)
- **Chain-of-thought** → ch05 (introduced, p. 405), ch06 (insufficient alone for planning, p. 504), ch08 (CoT training data), ch09 (inflates user-visible TTFT)

## Prerequisite Chains
- ch01 → ch02 → ch09 (completion machine → prefill/decode + KV cache → inference optimization)
- ch02 → ch07 → ch08 (post-training concepts → finetuning technique → the data that feeds it)
- ch03 → ch04 → ch10 (evaluation methods → application pipelines → production monitoring/feedback)
- ch05 → ch06 → ch07 (adaptation in increasing cost; exhaust each rung before the next)
- ch03 → ch08 (judge + similarity methods must be understood before synthetic-data verification)
- ch02 → ch05 (sampling + injection root cause before prompt engineering and its defenses)

## Framework Combinations
- **RAG (ch06)** + **Finetuning (ch07)**: diagnose failure type — information-based → RAG, behavioral → finetune; both → RAG first (p. 559)
- **Data synthesis (ch08)** + **AI judge / functional verification (ch03-04)**: only verified synthetic data self-improves (Llama 3's pipeline, p. 678-688)
- **LoRA** + **Quantization** = QLoRA (ch07); quantization reappears serving-side (ch09) — same bits, two budgets
- **Latency decomposition (TTFT/TPOT)** + **Goodput** (ch09): tune batching only until SLO-meeting throughput stops rising
- **Instruction hierarchy (ch05)** + **System guardrails (ch10)**: model-, prompt-, and system-level defense in depth
- **Evaluation-driven development** + **Four-step model selection** (ch04): criteria before building; public benchmarks only filter, your pipeline picks
- **Data flywheel (ch08)** + **Feedback design (ch10)** + **Preference finetuning (ch02/ch07)**: conversational signals → preference pairs → aligned model
- **Crawl-Walk-Run (ch01)** + **Human-in-the-loop write actions (ch06/ch10)**: automation level rises only as measured trust accrues
