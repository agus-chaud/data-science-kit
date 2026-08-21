# Chapter 1: Introduction to Building AI Applications with Foundation Models

## Core Idea
Scale turned language models into general-purpose foundation models available as a service, which collapsed the barrier to building AI applications — AI engineering is the resulting discipline of building on top of readily available models instead of training your own, shifting the work from modeling/training toward adaptation and evaluation (p. 29, p. 86).

## Frameworks Introduced
- **Self-supervision** (p. 36-37): the model infers labels from the input data itself — each text sequence supplies both contexts and the tokens to predict. This is what broke the data-labeling bottleneck (ImageNet's 1M labels cost ~$50K; scaling to 1M categories would cost ~$50M) and let language models scale into LLMs. Distinct from unsupervised learning, which needs no labels at all (p. 38).
  - When to use this lens: explaining why LLMs (and CLIP-style multimodal models via "natural language supervision", p. 41) scaled where supervised models couldn't.
- **Three adaptation techniques** (p. 43): **prompt engineering** (detailed instructions + examples), **RAG** (retrieval-augmented generation — supplement instructions with a database), **finetuning** (further train on your data). Split into two categories by whether they update model weights: prompt-based (no weight update, less data, easier to start, lets you try more models) vs. finetuning (weight update, more data and complexity, but can significantly improve quality/latency/cost and handle tasks the model never saw) (p. 87).
- **Use Case Evaluation — three risk levels** (p. 69): (1) existential — competitors with AI make you obsolete (7% of Gartner 2023 executives cited "business continuity"; build in-house), (2) missed profit/productivity opportunities (most companies; buy options often win), (3) FOMO/R&D — unsure where AI fits but can't be left behind.
- **Apple's role-of-AI axes** (p. 70-71): **Critical or complementary** (the more critical AI is, the more accurate/reliable it must be — Face ID vs. Smart Compose); **Reactive or proactive** (reactive needs low latency; proactive can be precomputed but needs a higher quality bar because users didn't ask for it); **Dynamic or static** (continually updated per-user vs. periodically updated shared model).
- **Crawl-Walk-Run** (Microsoft 2023) (p. 73): 1. Crawl — human involvement mandatory; 2. Walk — AI interacts directly with internal employees; 3. Run — increased automation, potentially direct AI interaction with external users. Promote as acceptance rate rises (e.g., 95% of AI-suggested responses used verbatim → let AI answer simple requests directly).
- **Three types of competitive advantage in AI** (p. 74): technology, data, and distribution. With foundation models, core technology is similar across companies and distribution belongs to big players — a startup's moat is getting to market first and gathering usage data ("data flywheel").
- **Three layers of the AI stack** (p. 81-82): **application development** (prompts + context, evaluation, interfaces — most action post-2022), **model development** (modeling/training, dataset engineering, inference optimization), **infrastructure** (serving, data/compute management, monitoring — least changed).
- **AI engineering vs. ML engineering — three differences** (p. 86): (1) you use models others trained → less modeling, more model adaptation; (2) models are bigger, more compute-hungry, higher latency → more inference optimization and GPU/cluster skills; (3) outputs are open-ended → evaluation becomes a much bigger problem.

## Key Concepts
- **Token**: basic unit of a language model — character, word, or sub-word; for GPT-4, ~3/4 of a word, so 100 tokens ≈ 75 words (p. 31-32).
- **Masked language model**: predicts missing tokens using context before and after (BERT); suited to non-generative tasks like classification and code debugging (p. 33).
- **Autoregressive language model**: predicts the next token from preceding tokens only; the default for text generation and what "language model" means in this book (p. 33-34).
- **Foundation model**: a model that can be built upon for different needs; covers both LLMs and large multimodal models (LMMs); marks the shift from task-specific to general-purpose models (p. 40, p. 42).
- **Completion machine**: a language model given a prompt tries to complete it — translation, summarization, coding, classification can all be framed as completion, but completion is not conversation (p. 35-36).
- **Embedding model**: produces vectors capturing the meaning of data (e.g., CLIP's joint text-image embeddings); CLIP is not generative, but multimodal embedding models are the backbones of generative multimodal models (p. 42).
- **Human-in-the-loop**: involving humans in AI's decision-making processes (p. 72).
- **Model as a service**: models exposed via APIs, removing the need to host/serve models yourself — the key enabler of the low entry barrier (p. 46).
- **Pre-training / finetuning / post-training** (p. 89-90): pre-training = from randomly initialized weights (up to 98% of InstructGPT's compute+data); finetuning = continuing training from existing weights; post-training = conceptually the same as finetuning but done by model developers rather than application developers. Prompt engineering is NOT training — feeding journal entries into ChatGPT via context is prompting, not finetuning (p. 91).
- **Inference**: computing an output given an input; autoregressive generation is sequential (10 ms/token → 1 s for 100 tokens), which is why hitting the ~100 ms latency users expect is hard (p. 77, p. 92).

## Mental Models
- Think of a language model as a completion machine: frame your task (translate, classify, summarize) as "text to be completed" and remember completions are probabilistic predictions, not guaranteed answers (p. 35).
- Use the buy-vs-build lens: adapting an existing model ≈ "ten examples and one weekend" vs. building from scratch ≈ "1 million examples and six months" — default to adapt; build task-specific only when smaller/faster/cheaper matters or AI is existential to your business (p. 43, p. 70).
- Expect the last-mile challenge: demo-to-product is where the cost lives. "The journey from 0 to 60 is easy, whereas progressing from 60 to 100 becomes exceedingly challenging" (UltraChat); LinkedIn hit 80% of desired experience in one month, then needed four more months to pass 95% (p. 76).
- Start internal-facing and close-ended: enterprises deploy internal apps (knowledge management) before external ones (customer chatbots) because risks are lower, and close-ended tasks like classification are easier to evaluate (p. 56).
- With foundation models, ship product first: the new workflow inverts traditional ML — build the product, then invest in data and models once it shows promise ("The Rise of the AI Engineer", Shawn Wang) (p. 99).

## Anti-patterns
- **Confusing prompting with training**: teaching a model via context input is prompt engineering, not training or finetuning — using the terms interchangeably muddies effort/cost estimates (p. 91).
- **Building a thin wrapper on an assumed model weakness**: e.g., a PDF-parsing app on the assumption ChatGPT can't parse PDFs — when the underlying model improves, your layer is subsumed (p. 73-74).
- **Trusting a demo as a product forecast**: impressive base capabilities make demos cheap; hallucinations and product kinks consume months afterward (p. 76).
- **Ignoring the pace of change in build/buy decisions**: in-house may look cheaper until providers halve prices three months later; a third-party provider may fold; regulation (GDPR-style compliance ~$9B, GPU export controls, unresolved IP questions) can invalidate choices overnight (p. 78).
- **Comparing models under unequal prompting**: Google reported Gemini Ultra beating GPT-4 on MMLU using CoT@32 (32 examples) vs. GPT-4's 5-shot; at equal 5-shot, GPT-4 won (86.4% vs. 83.7%) — evaluation results are inseparable from the adaptation technique used (p. 94-95).

## Reference Tables

**Common generative AI use cases** (Table 1-3, p. 54-55) — from 205 open source apps (500+ stars) + 50 enterprise interviews:

| Category | Consumer | Enterprise |
|---|---|---|
| Coding | Coding | Coding |
| Image/video production | Photo/video editing, design | Presentations, ad generation |
| Writing | Email, social/blog posts | Copywriting, SEO, reports, memos |
| Education | Tutoring, essay grading | Onboarding, upskill training |
| Conversational bots | General chatbot, AI companion | Customer support, product copilots |
| Information aggregation | Summarization, talk-to-your-docs | Summarization, market research |
| Data organization | Image search, memex | Knowledge management, document processing |
| Workflow automation | Travel/event planning | Data extraction/entry/annotation, lead generation |

**Model development: traditional ML vs. foundation models** (Table 1-4, p. 93):

| Category | Traditional ML | Foundation models |
|---|---|---|
| Modeling and training | ML knowledge required | ML knowledge nice-to-have (disputed) |
| Dataset engineering | Feature engineering on tabular data | Deduplication, tokenization, context retrieval, quality control on unstructured data |
| Inference optimization | Important | Even more important |

**App development: traditional ML vs. foundation models** (Table 1-6, p. 98):

| Category | Traditional ML | Foundation models |
|---|---|---|
| AI interface | Less important | Important |
| Prompt engineering | Not applicable | Important |
| Evaluation | Important | More important |

## Worked Example
**Planning a customer support chatbot end-to-end** (reconstructed from p. 69-76):

1. **Why build it** — classify the risk level: most likely level 2 (boost productivity/retention), so weigh buy options seriously before building (p. 69-70).
2. **Role of AI** — complementary (support works without it), reactive (needs speed), likely static at first. Pick the human-in-the-loop mode: (a) AI drafts responses agents reference, (b) AI answers simple requests and routes complex ones, or (c) AI answers everything directly (p. 71-72).
3. **Automation ramp** — apply Crawl-Walk-Run: start with AI-suggested drafts (Crawl); if agents use 95% of suggestions for simple requests verbatim, promote those to direct AI responses (Run for that slice) (p. 73).
4. **Set expectations** — business metrics: % of messages automated, extra throughput, response-time improvement, human labor saved — plus customer satisfaction, since answering more messages doesn't equal happier users (p. 75). Usefulness threshold: quality metrics, latency (TTFT, TPOT, total — if humans currently take a median of an hour, anything faster may suffice), cost per inference request, interpretability/fairness (p. 76).
5. **Milestone planning** — evaluate off-the-shelf models first: if the goal is automating 60% of tickets and the base model already handles 30%, the gap defines the work; evaluation may kill the project if reaching the threshold costs more than the return (p. 76).
6. **Maintenance** — budget for riding the "bullet train": model/API swaps require reworking prompts and workflows, so versioning and evaluation infrastructure must exist before you need them (p. 78).

## Key Takeaways
1. Frame any task as completion first — translation, classification, summarization all reduce to it — but design for probabilistic, sometimes-wrong outputs (p. 35-36).
2. Choose adaptation by cost gradient: prompt engineering → RAG → finetuning, in increasing order of data and complexity; scratch-training generally needs more data than finetuning, which needs more than prompting (p. 43, p. 92).
3. Before building, answer three questions: why (risk level), what role AI/humans play (critical-reactive-dynamic axes + human-in-the-loop mode), and what defends the product (technology/data/distribution — usually data) (p. 69-74).
4. Define a usefulness threshold (quality, latency, cost) before shipping, and expect the 60→100 stretch to dwarf the 0→60 demo (p. 76).
5. AI engineering = less model development, more adaptation and evaluation; evaluation is the discipline's hardest and fastest-growing problem because outputs are open-ended (p. 86, p. 94).
6. Enduring ML principles still apply: map business metrics to ML metrics, experiment systematically (now over models/prompts/retrieval/sampling instead of hyperparameters), and close the production feedback loop (p. 84).
7. Full-stack skills gain value: the new workflow rewards fast product iteration, JS tooling is rising (LangChain.js, Transformers.js, Vercel AI SDK), and AI engineers are far more involved in product than ML engineers were (p. 98-99).

## Connects To
- **Ch 2**: how the completion machine actually works — training data, architecture, post-training, and sampling explain the probabilistic outputs assumed here; pre-training vs. post-training quantified (post-training ≈ 2% of compute).
- **Ch 3**: the "evaluation is the hardest problem" claim becomes a methodology — perplexity, exact evaluation, AI judges, comparative evaluation.
- **Ch 4**: usefulness thresholds and business-metric mapping become evaluation-driven development and the four-step model selection workflow.
- **Ch 5**: prompt engineering — rung one of the adaptation ladder introduced at p. 43.
- **Ch 6**: RAG and agents — rung two; context construction for the completion machine.
- **Ch 7**: finetuning — rung three; the buy-vs-build cost gradient made concrete ("ten examples and one weekend" quantified).
- **Ch 8**: the data moat / flywheel (p. 74) becomes dataset engineering — data as the differentiator when models commoditize.
- **Ch 9**: the sequential-decoding latency problem (10 ms/token, p. 92) is exactly what inference optimization attacks.
- **Ch 10**: Crawl-Walk-Run automation levels, business metrics, and the feedback loop become architecture design and user-feedback engineering.
- **External**: Eloundou et al. 2023 "GPTs are GPTs" (occupation exposure), Super-NaturalInstructions benchmark (Wang et al. 2022), MMLU (Hendrycks et al. 2020), CLIP (OpenAI 2021), "The Rise of the AI Engineer" (Shawn Wang 2023).
