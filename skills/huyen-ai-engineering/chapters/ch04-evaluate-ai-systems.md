# Chapter 4: Evaluate AI Systems

## Core Idea
Evaluation is the biggest bottleneck to AI adoption: define application-specific evaluation criteria before building (evaluation-driven development), use public benchmarks only to narrow the candidate pool, then rely on your own evaluation pipeline — because public benchmarks are likely contaminated and never represent your application (p. 284, 346).

## Frameworks Introduced
- **Evaluation-driven development** (p. 283): define evaluation criteria before building, inspired by test-driven development. The most common enterprise AI apps in production (recommenders, fraud detection, coding) are the ones with clear evaluation criteria (p. 283).
  - When to use: before investing in any AI application.
  - How: start with a criteria list in four buckets — domain-specific capability, generation capability, instruction-following capability, cost and latency (p. 284).
- **Four-step model selection workflow** (p. 318): (1) filter out models whose hard attributes (license, privacy policy, size) don't work for you; (2) use public benchmark/leaderboard data to shortlist promising models balancing quality, latency, cost; (3) run your own evaluation pipeline to pick the winner; (4) monitor in production and iterate. Steps are iterative, not linear (p. 319).
  - When to use: every time you revisit model choice — which happens repeatedly as you move through prompting, finetuning, etc. (p. 316).
- **Hard vs. soft attributes** (p. 317): hard = impossible/impractical to change (license, training data, your privacy policy); soft = improvable (accuracy, toxicity, factual consistency). Latency is soft if you host the model, hard if someone else does (p. 318).
- **Local vs. global factual consistency** (p. 288–289): verify output against a provided context (local — summarization, support bots, RAG) or against open knowledge (global — general chatbots, fact-checking). Local is far easier; the hardest part of global verification is determining what the facts are (p. 289).
- **SAFE — Search-Augmented Factuality Evaluator** (Google DeepMind, p. 293): 4 steps — decompose response into individual statements; revise each to be self-contained; propose fact-checking queries to a search API; use AI to judge consistency with search results (p. 293).
- **Self-verification / SelfCheckGPT** (p. 292): generate N new responses; if they disagree with the original, it's likely hallucinated. Works but is prohibitively expensive (p. 292–293).
- **Three-step evaluation pipeline design** (p. 351–366): (1) evaluate all components — per task, per turn, per intermediate output; (2) create an evaluation guideline with criteria, scoring rubrics, and ties to business metrics; (3) define evaluation methods and data, then evaluate the pipeline itself. See Worked Example.

## Key Concepts
- **Factual consistency**: whether output is supported by a given context (local) or open knowledge (global); the most-demanded hallucination metric (p. 287–289).
- **Textual entailment (NLI)**: framing consistency checking as classifying (premise, hypothesis) into entailment / contradiction / neutral — entailment implies consistency, contradiction implies inconsistency (p. 294–295).
- **Instruction-following capability**: whether the model obeys format/content constraints (JSON, word limits, keyword rules) — distinct from knowing how to do the task (p. 301–303). Benchmarks: IFEval (25 automatically verifiable instruction types), INFOBench (yes/no criteria checked by AI judge) (p. 303, 307).
- **Public leaderboards**: rankings aggregating a small benchmark subset (HF Open LLM: 6 benchmarks; HELM Lite: 10, only 2 shared) — selection and aggregation are opaque, so a high rank far from guarantees fit for your app (p. 338–342).
- **Benchmark contamination** (data contamination / training on the test set): the model was trained on its evaluation data; a 1M-parameter model trained on test sets achieved near-perfect scores ("Pretraining on the Test Set Is All You Need", p. 346). Mostly unintentional via internet scraping (p. 347).
- **Model selection**: you don't care which model is best — you care which is best *for your application*; it's akin to building a private leaderboard from your criteria (p. 316, 343).
- **Inference service / model API**: the service that hosts and runs the model vs. the interface users call; same model via different APIs can perform differently (p. 323–324).
- **Open weight vs. open model**: open weight = public weights without training data; open model = weights + open data (p. 320).
- **Turn-based vs. task-based evaluation**: quality of each output vs. whether the task got done and in how many turns; task-based matters more but task boundaries are fuzzy (p. 352–353).

## Mental Models
- Use **the lamppost test** when scoping projects: evaluating only what's easy to measure is searching for the lost key under the lamppost — reliable eval pipelines for hard-to-measure tasks unlock new applications (p. 283–284).
- Think of **model selection as a repeated two-step**: find the best achievable performance, then map models on cost-performance axes and pick the best value (p. 317).
- Use **must-have vs. nice-to-have** for latency: users always say yes to faster, but high latency is often an annoyance, not a deal breaker — filter on non-negotiables first (Pareto optimization, p. 312–313).
- Use **"if you care about something, put a test set on it"**: slices for known failure modes, typos, out-of-scope inputs, user tiers (p. 362–363).

## Anti-patterns
- **Deploying without evaluation**: worse than never deploying — it costs to maintain and you can't tell if it helps or hurts (p. 282).
- **Trusting MCQ benchmarks for generation tasks**: multiple-choice tests discrimination, not generation; answers flip with a stray space or "Choices:" phrase (p. 286).
- **Trusting public leaderboard rank as fitness for your app**: benchmarks are contaminated, correlated (WinoGrande/MMLU/ARC-C at ~0.87–0.90 Pearson), and averaged with equal weights arbitrarily (p. 341–342).
- **Correct-but-bad responses**: LinkedIn's "You are a terrible fit" was correct but unhelpful — a guideline must define *good*, not just *correct* (p. 355).
- **Skipping evaluation to cut latency**: "a risky bet" (p. 366).
- **Ignoring Simpson's paradox**: model A can beat B on every subset yet lose overall — slice your data (p. 361–362).

## Code Examples
```text
Factual Consistency: Does the summary untruthful or misleading facts
that are not supported by the source text?

Source Text:
{{Document}}

Summary:
{{Summary}}

Does the summary contain factual inconsistency?
Answer:
```
- **What it demonstrates**: the AI-judge prompt Liu et al. (2023) used for factual consistency of summaries — note it ships with a typo, copied verbatim, showing how easily prompt errors slip in (p. 291–292, 369).

## Reference Tables

**Host-yourself vs. model API — 7 axes** (p. 325–336; Table 4-4, p. 334):

| Axis | Model APIs | Self-hosting |
|---|---|---|
| Data privacy | Data leaves org (Samsung/ChatGPT leak, p. 325–326); provider policies can change (Zoom) | Data stays internal |
| Data lineage/copyright | Contracts may shield you legally | Fewer checks; open data is inspectable in theory, impractical at scale (p. 326) |
| Performance | Best models stay closed; gap closing but incentives keep strongest proprietary (p. 327–328) | Best open models lag a bit |
| Functionality | Scaling, function calling, structured outputs out of the box; logprobs rarely exposed; finetuning only if provider allows (p. 329–330) | Full access to logprobs/intermediates; you build functionality yourself |
| Cost | Per-token API cost; roughly flat per token at scale | Engineering talent + time; per-token cost drops with scale (p. 314) |
| Control/access/transparency | Rate limits, over-censoring guardrails (Convai's "As an AI model, I don't have physical abilities"), unannounced model changes, risk of losing access (p. 332) | Can freeze the model; easier to inspect changes |
| On-device | Impossible | Possible, though hard (p. 333) |

**Example model-selection criteria table** (Table 4-3, p. 315):

| Criteria | Metric | Benchmark | Hard requirement | Ideal |
|---|---|---|---|---|
| Cost | Cost per output token | — | < $30.00/1M tokens | < $15/1M |
| Scale | TPM (tokens per minute) | — | > 1M TPM | > 1M |
| Latency | Time to first token (P90) | Internal user prompts | < 200ms | < 100ms |
| Latency | Time per query (P90) | Internal user prompts | < 1m | < 30s |
| Overall quality | Elo score | Chatbot Arena | > 1200 | > 1250 |
| Code generation | pass@1 | HumanEval | > 90% | higher |
| Factual consistency | Internal GPT metric | Internal hallucination set | > 0.8 | > 0.9 |

**Eval-set sizing** (OpenAI estimate, Table 4-7, p. 364): score difference to detect → samples for 95% confidence: 30% → ~10; 10% → ~100; 3% → ~1,000; 1% → ~10,000. Rule: 3× smaller difference needs 10× more samples.

**Contamination detection** (p. 348–349): n-gram overlap (13-token overlap → "dirty" sample; accurate but needs training-data access) vs. perplexity (unusually low on eval data → likely seen; cheaper, less accurate).

## Worked Example
The author's pipeline construction for an open-ended application (p. 351–366), reconstructed:

**Step 1 — Evaluate every component** (p. 351). App: extract current employer from a resume PDF, in two steps: (a) PDF → text, (b) text → employer. Evaluate (a) with similarity to ground-truth text, (b) with accuracy given correct text — otherwise you can't localize failures. Also evaluate per turn (each output's quality) and per task (did the debugging chatbot fix the bug, and in 2 turns or 20?) — task-based matters more (p. 352–353). Example: BIG-bench `twenty_questions`, scored on whether and how fast one model instance guesses the other's concept (p. 353).

**Step 2 — Create the evaluation guideline** (p. 354). Define what the app should NOT do (should a support bot answer election questions?). Define *good*, not just *correct* (LinkedIn's job-fit lesson, p. 355). Pick 2–3 criteria per app (LangChain: users average 2.3) — e.g., support bot: relevance, factual consistency, safety (p. 355–356). For each criterion choose a scoring system (binary; or -1/0/1 entailment) and write a rubric with example-scored responses; validate the rubric on humans until unambiguous (p. 356). Tie metrics to business outcomes: "factual consistency 80% → automate 30% of requests; 90% → 50%; 98% → 90%", and set the usefulness threshold below which the app is unusable (p. 357–358).

**Step 3 — Methods and data** (p. 358). Mix methods per criterion: small toxicity classifier for safety, semantic similarity for relevance, AI judge for factual consistency; layer a cheap classifier on 100% of traffic with an expensive AI judge on 1% (p. 359). Use logprobs when available for confidence and perplexity signals; keep human evaluation as the North Star (LinkedIn manually reviews up to 500 daily conversations, p. 360). Annotate eval data from production; slice it (tiers, traffic source, known-failure sets, typo sets, out-of-scope sets) to debug, avoid bias, and dodge Simpson's paradox (p. 361–362). Check set size by bootstrapping: resample your 100 examples with replacement; 90% on one bootstrap but 70% on another means the set is too small (p. 363). Finally, evaluate the pipeline itself: right signals? reproducible (judge temperature 0)? metrics uncorrelated enough? acceptable added cost/latency? Track every experiment variable — data, rubric, judge prompts, sampling configs (p. 365–366).

## Key Takeaways
1. Define evaluation criteria before building — an app you can't evaluate is worse than one never deployed (p. 282–283).
2. Bucket criteria into domain-specific, generation, instruction-following, and cost/latency; evaluate each with the cheapest method that gives a trustworthy signal (p. 284).
3. Use public benchmarks and leaderboards only to filter out bad models, never to pick the winner — contamination and opaque aggregation make them untrustworthy for final selection (p. 346, 350).
4. Decide build-vs-buy along the seven axes (privacy, lineage, performance, functionality, cost, control, on-device); revisit at each scale change since API cost is flat per token but self-host cost drops with volume (p. 314, 324–333).
5. Curate your own instruction benchmark — if you need YAML output, put YAML instructions in your eval set (p. 308).
6. Write scoring rubrics with examples and validate them on humans; ambiguous guidelines produce misleading scores (p. 354–356).
7. Bootstrap your eval set to check reliability, and size it by the score difference you must detect (10% diff ≈ 100 samples at 95% confidence) (p. 363–364).

## Connects To
- **Ch 1**: usefulness thresholds and business-metric mapping from project planning become concrete criteria tables and thresholds here.
- **Ch 2**: hallucination causes explain why factual consistency is the top criterion; perplexity-based contamination detection and logprob access trace back to model mechanics.
- **Ch 3**: supplies the evaluation methods (exact eval, AI judge, perplexity) this chapter assembles into application pipelines (p. 280).
- **Ch 5**: instruction-following capability is the prerequisite for prompting to work; MCQ prompt sensitivity shows why benchmarks are fragile.
- **Ch 6**: local factual consistency is the crucial criterion for RAG; retrieval and agent evaluation extend the component-level pipeline.
- **Ch 7**: run this evaluation pipeline before and after finetuning; model selection recurs at every adaptation step.
- **Ch 8**: evaluation guidelines are the same artifact as annotation guidelines; annotated production data and slices feed finetuning and synthesis (p. 356, 360).
- **Ch 9**: the cost/latency criteria (TTFT, TPOT, per-token pricing) and the API-vs-self-host cost axes are explained by inference mechanics and optimization.
- **Ch 10**: production monitoring, drift detection, and output guardrails extend this pipeline into the live system.
- **Test-driven development**: the software-engineering analogue that names evaluation-driven development (p. 283).
