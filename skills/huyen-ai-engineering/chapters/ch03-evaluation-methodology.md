# Chapter 3: Evaluation Methodology

## Core Idea
Open-ended foundation models can't be graded against a closed set of ground truths, so AI engineering needs a layered evaluation toolkit — language modeling metrics (perplexity, cross entropy) as capability proxies, exact methods (functional correctness, similarity scores) where possible, and subjective methods (AI as a judge, comparative evaluation) everywhere else — with each method's biases and failure modes understood before you trust its numbers (p. 206-207).

## Frameworks Introduced
- **Language modeling metrics family: cross entropy, perplexity, BPC, BPB** (p. 212): four interconvertible measures of how well a model predicts a token sequence — know one plus tokenization details and you can compute the rest.
  - When to use: comparing base-model capability, detecting data contamination in benchmarks, deduplicating training data, flagging abnormal text (p. 220-222).
  - How: cross entropy H(P,Q) = H(P) + D_KL(P||Q); perplexity PPL = 2^H (bits) or e^H (nats). Compute a text's perplexity from the logprobs the model assigns each token given its predecessors (p. 215-217, 223).
- **Exact evaluation: functional correctness** (p. 224): judge a system by whether it does what it's supposed to — execution accuracy for code, game score for bots, energy saved for schedulers.
  - When to use: any task with a measurable objective; the ultimate metric when available.
  - How: run generated code against test cases; report **pass@k** — the fraction of problems where any of k generated samples passes all test cases (used by HumanEval, MBPP, Spider, BIRD-SQL) (p. 225-226).
- **Similarity measurements against reference data** (p. 227): score a generated response by closeness to reference responses (ground truths / canonical responses).
  - When to use: tasks with curated references (translation, QA) where functional correctness can't be automated.
  - How: four options — evaluator judgment, exact match, lexical similarity (n-gram overlap, edit distance: BLEU, ROUGE, METEOR++, TER, CIDEr), semantic similarity (cosine similarity between embeddings: BERTScore, MoverScore) (p. 228-235).
- **AI as a judge** (p. 239): use an AI model plus a prompt to score responses — fast, cheap, reference-free, works in production; the most common production evaluation method as of the book's writing (58% of evaluations on LangChain's platform in 2023) (p. 239-240).
  - When to use: open-ended criteria (relevance, faithfulness, toxicity) with no reference data; production guardrails; when human evaluation is too slow/expensive.
  - How: prompt must specify (1) the task, (2) the evaluation criteria in detail, (3) the scoring system — classification, discrete numeric, or continuous numeric. Include scored examples (p. 244-245).
- **Comparative evaluation** (p. 262-263): rank models from pairwise preference matches instead of independent (pointwise) scores; a rating algorithm (Elo, Bradley-Terry, TrueSkill) converts win rates into a ranking (p. 266-267).
  - When to use: subjective quality where "which is better" is easier than "how good is this"; powers LMSYS Chatbot Arena.
  - How: show evaluators two responses side by side, record the winner (ties allowed), fit ratings; ranking quality = how well it predicts future match outcomes (p. 263-267).
- **Specialized judges: reward models, reference-based judges, preference models** (p. 259-261): small trained judges that can beat general-purpose LLM judges for specific judgments.

## Key Concepts
- **Entropy**: average information per token; lower entropy = more predictable language (p. 213-214).
- **Cross entropy**: how hard it is for a model to predict a dataset; equals data entropy plus the KL divergence of the learned distribution from the true one; not symmetric (p. 215-216).
- **Perplexity**: exponential of cross entropy — the amount of uncertainty (effective branching factor) when predicting the next token (p. 217).
- **Bits-per-byte (BPB)**: cross entropy normalized per byte of original text, comparable across tokenizers; also the model's text-compression efficiency (p. 216).
- **Functional correctness / execution accuracy**: does the output actually work when executed against test cases (p. 224).
- **pass@k**: fraction of problems solved when any of k samples passes all tests; pass@1 ≤ pass@3 ≤ pass@10 in expectation (p. 226).
- **Reference-based vs. reference-free metrics**: metrics that require ground-truth responses vs. those that don't; reference-based evaluation is bottlenecked by reference-data generation (p. 227).
- **Embedding**: vector representation capturing meaning; typical size 100-10,000 (BERT base 768, text-embedding-3-large 3072); quality judged by whether similar texts get closer embeddings, benchmarked by MTEB (p. 235-237).
- **AI judge**: a system = model + prompt (+ sampling parameters); change any part and you have a different judge (p. 248).
- **Preference model**: takes (prompt, response 1, response 2), predicts which response users prefer — key for alignment data and evaluation (p. 261).

## Mental Models
- Think of perplexity as the number of options the model is choosing among: perplexity 4 means the model is as uncertain as a fair choice among 4 tokens (p. 217).
- Use perplexity as a training-data detector: low perplexity on a benchmark means the benchmark likely leaked into training data, so benchmark scores are inflated; high perplexity on new data means it's worth adding (deduplication) (p. 222).
- Treat evaluation methods as a fallback chain: functional correctness when the task has a measurable objective → similarity against references when you have reference data → AI as a judge when you have neither (p. 224-240).
- Use comparative evaluation when scoring is harder than choosing: for subjective quality (and for superhuman outputs), humans who can't assign a score can still pick a winner (p. 263, 274).

## Anti-patterns
- **Eyeballing / vibe-check evaluation**: ad hoc go-to prompts based on the curator's experience rather than the application's needs — survives project kickoff, fails at iteration (p. 210).
- **Trusting perplexity on post-trained models**: SFT/RLHF teach task completion and typically *raise* perplexity ("post-training collapses entropy"); quantization also shifts it unpredictably (p. 221).
- **Optimizing lexical similarity as if it were quality**: on HumanEval, BLEU scores for wrong and correct solutions were similar; good responses score low when the reference set is incomplete (Fuyu's captions) or wrong (WMT 2023 references) (p. 231-232).
- **Comparing scores across judges**: MLflow (1-5), Ragas (0/1), and LlamaIndex (YES/NO) all measure "faithfulness" with different prompts and scales — scores aren't comparable, and criteria aren't standardized (p. 250-252).
- **Using an unversioned judge as a monitoring baseline**: if the judge's prompt or model changes between months, score movement tells you nothing about your application. "Do not trust any AI judge if you can't see the model and the prompt" (p. 253-254).
- **Ignoring judge biases**: self-bias (GPT-4 favors itself by ~10% win rate, Claude-v1 by ~25%), first-position bias (opposite of human recency bias), verbosity bias (longer-but-wrong beats shorter-but-correct) (p. 256-257).
- **Preference voting on correctness questions**: asking users to pick between "Yes" and "No" on factual questions produces misaligned training signal and frustrates users who asked because they don't know the answer (p. 264).
- **Reading a win rate as absolute performance**: "B beats A 51% of the time" doesn't say whether either is good enough, or what ticket-resolution boost to expect — comparative results can't drive cost-benefit analysis alone (p. 273-274).

## Code Examples
```text
Perplexity of model X on token sequence [x1, x2, ..., xn]:
  PPL(X) = product over i of P(xi | x1..xi-1), raised to (-1/n)
  (equivalently: 2^H(P,Q) in bits, e^H(P,Q) in nats)
Requires the model's logprobs per token — not all commercial APIs expose them.
```
- **What it demonstrates**: how to compute a text's perplexity from next-token probabilities, and why logprob access matters when choosing an API (p. 223).

## Reference Tables
Language modeling metrics (p. 212-217):

| Metric | Definition | Interpretation |
|---|---|---|
| Entropy H(P) | Avg. information per token of the data | Lower = more predictable language |
| Cross entropy H(P,Q) | H(P) + D_KL(P\|\|Q) | How hard the model finds predicting the data; perfect learning → H(P,Q) = H(P) |
| BPC / BPB | Cross entropy per character / per byte | Tokenizer-independent comparison; BPB 3.43 → compresses text to <half its size |
| Perplexity | 2^H (bits) or e^H (nats) | Effective number of next-token options; values ≈3 or lower are common |

Perplexity rules of thumb (p. 218-220): more structured data → lower expected PPL; bigger vocabulary → higher PPL; longer context → lower PPL. Larger GPT-2 models give consistently lower perplexity (LAMBADA PPL: 35.13 at 117M → 8.63 at 1542M) (p. 221).

Judge scoring-system guidance (p. 245): classification > discrete numeric > continuous numeric; discrete scales wider than 1-5 degrade quality; include scored examples (raised GPT-4 judge consistency from 65% to 77.5% but quadrupled cost — Zheng et al.) (p. 249).

Built-in judge criteria by tool (p. 243): Azure AI Studio (groundedness, relevance, coherence, fluency, similarity), MLflow (faithfulness, relevance), LangChain Criteria Evaluation (conciseness, correctness, harmfulness, ...), Ragas (faithfulness, answer relevance) — none standardized across tools.

Specialized judges (p. 260-261):

| Judge type | Input | Output | Example |
|---|---|---|---|
| Reward model | (prompt, response) | Quality score | Cappy (Google, 360M params, 0-1 score) |
| Reference-based | (candidate, reference[, rubric]) | Similarity/quality score | BLEURT; Prometheus (1-5, reference = 5) |
| Preference model | (prompt, response 1, response 2) | Which response users prefer | PandaLM, JudgeLM |

## Worked Example
Designing an AI-as-a-judge prompt, reconstructed from Azure AI Studio's relevance judge (p. 246-248). The three prompt archetypes (p. 241-242): (1) score a response alone given the question, (2) compare a generated response to a reference, (3) compare two responses and pick a winner (useful for preference data and comparative evaluation). Azure's relevance prompt is type 2 and contains every required element:

1. **Task**: "Score the relevance between a generated answer and the question based on the ground truth answer in the range between 1 and 5, and please also provide the scoring reason."
2. **Criteria, detailed**: "Your primary focus should be on determining whether the generated answer contains sufficient information to address the given question according to the ground truth answer" — plus explicit rules like "if the generated answer contradicts the ground truth answer, it will receive a low score of 1-2."
3. **Scoring system**: discrete 1-5 (the empirically favored format).
4. **Scored example with justification**: question "Is the sky blue?", ground truth "Yes, the sky is blue", generated "No, the sky is not blue" → score 1-2, reason: contradiction with ground truth.

Why each choice matters: discrete 1-5 because LLMs judge better with text/classes than numbers and degrade on wide or continuous scales (p. 245); the in-prompt example because examples raise consistency (65% → 77.5% for GPT-4, at 4x the cost) (p. 249); the explanation requirement because rationales make judgments auditable (p. 240-241). Finally, version the whole judge: model + prompt + sampling parameters together define it — change any one and last month's 90% coherence isn't comparable to this month's 92% (p. 248, 253).

## Key Takeaways
1. Invest in systematic evaluation early — design it around where your system is likely to fail; benchmarks saturate fast (GLUE → SuperGLUE in a year, MMLU → MMLU-Pro) and word-of-mouth doesn't scale (p. 206-208).
2. Use perplexity as a base-model capability proxy and data tool (contamination detection, deduplication, abnormality detection), but not on post-trained or quantized models (p. 220-222).
3. Prefer functional correctness whenever the task has a measurable objective; it's the only metric that directly measures whether the application does its job (p. 224).
4. When using similarity scores, budget for reference-data quality — incomplete or wrong references silently punish good responses; semantic similarity needs fewer references than lexical but inherits the embedding model's flaws (p. 231-234).
5. An AI judge = model + prompt + sampling parameters. Pin and disclose all three, supplement with exact and/or human evaluation, and never compare scores across differently configured judges (p. 248, 253-257).
6. Use comparative evaluation to pick between models on subjective quality, but pair it with absolute measures — a win rate can't tell you whether a model is good enough or worth its cost (p. 262-263, 273-274).
7. Only collect preference votes from evaluators competent on the question; preference signals on correctness questions or from unknowledgeable voters pollute rankings (p. 264, 270-272).

## Connects To
- **Ch 1**: open-ended outputs made evaluation AI engineering's hardest problem — this chapter is the methodological answer; entropy/perplexity build on ch01's language-modeling framing.
- **Ch 2**: post-training (SFT/RLHF) explains why perplexity degrades as a proxy; sampling variables govern judge consistency; comparison data links to preference finetuning.
- **Ch 4**: these methods (exact eval, AI judge, perplexity) get assembled into application pipelines and the model-selection workflow.
- **Ch 5**: self-critique is this chapter's self-eval concept applied at generation time; judge prompts are prompts — anatomy, versioning, and pinning rules apply to them.
- **Ch 6**: embeddings (MTEB) power semantic retrieval; AI judges validate agent plans and serve as reflection evaluators.
- **Ch 8**: similarity measurements and perplexity power deduplication and contamination checks; AI judges verify synthetic data and generate preference data.
- **Ch 10**: judge biases (position, verbosity) reappear as human feedback biases (position, preference/leniency); AI judges become production monitoring metrics.
- **External**: Elo/Bradley-Terry/TrueSkill rating systems from chess and gaming; MT-Bench, Chatbot Arena, HumanEval, MTEB benchmarks.
