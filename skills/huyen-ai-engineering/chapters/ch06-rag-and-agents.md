# Chapter 6: RAG and Agents

## Core Idea
A model's answer is only as good as the context you construct for each query. Two dominant patterns build that context: RAG retrieves relevant information from external memory sources, while the agentic pattern uses tools to gather information and act on the world — both backed by a memory system when information exceeds the context limit (p. 451).

## Frameworks Introduced
- **RAG (retrieval-augmented generation)** (p. 452): enhance generation by retrieving relevant information from external memory (internal DB, past chat sessions, the internet); coined by Lewis et al. (2020) for knowledge-intensive tasks where all knowledge can't fit in the model.
  - When to use: knowledge exceeds context limits, data changes often, or you need per-user data isolation. Anthropic's guidance: under ~200K tokens (~500 pages), just put the whole knowledge base in the prompt — no RAG needed (p. 454).
  - How: retriever (indexing + querying) fetches top chunks → post-process into final prompt → generator answers (p. 455-456).
- **Term-based retrieval (lexical retrieval)** (p. 459): rank by keyword statistics. TF-IDF: Score(D,Q) = Σ IDF(tᵢ) × f(tᵢ,D); IDF(t) = log(N/C(t)) (p. 460). BM25 normalizes TF by document length; Elasticsearch uses an inverted index (term → documents containing it) (p. 461-462).
- **Embedding-based retrieval (semantic retrieval)** (p. 462): embed query with the same model used at indexing, then vector search for the k nearest chunk embeddings in a vector database (p. 462-463). Prefer term-based vs embedding-based categorization over sparse vs dense — SPLADE uses sparse embeddings but behaves like dense retrieval (p. 458-459).
- **Hybrid search** (p. 472): combine term-based and embedding-based retrievers — sequentially (cheap retriever fetches candidates, expensive one reranks) or in parallel with **reciprocal rank fusion (RRF)**: Score(D) = Σ 1/(k + rᵢ(D)), k typically 60 (p. 473-474).
- **Agent** (p. 488): anything that perceives its environment and acts upon it — characterized by its environment and its set of actions, which its tool inventory augments. RAG systems are agents; retrievers and SQL executors are their tools (p. 489).
- **Agent task-solving loop** (p. 503): 1. Plan generation (task decomposition) → 2. Reflection and error correction on the plan → 3. Execution (function calling) → 4. Reflection and error correction on outcomes; loop until done. Decouple planning from execution — validate plans (heuristics or AI judges) before running them (p. 500).
- **ReAct** (Yao et al., 2022) (p. 517): interleave reasoning (planning + reflection) and action at every step: explain thinking → act → analyze observation, until the agent judges the task finished.
- **Reflexion** (Shinn et al., 2023) (p. 520): split reflection into an evaluator (scores the outcome) and a self-reflection module (analyzes what went wrong); agent proposes a new trajectory after each cycle.
- **Three memory mechanisms** (p. 532): internal knowledge (weights — for information all tasks need), short-term memory (context — fast, limited, current-task info), long-term memory (external retrievable data — persists across tasks, deletable without retraining).

## Key Concepts
- **Retriever**: component with two functions — indexing (process data for fast later retrieval) and querying (fetch data relevant to a query) (p. 455-456).
- **Vector search**: nearest-neighbor search over embeddings; naive k-NN is exact but slow, so large datasets use approximate nearest neighbor (ANN) algorithms (p. 463-464).
- **Context precision / context recall**: of retrieved documents, % relevant to the query / of relevant documents, % retrieved. Precision is cheap (AI judge on retrieved set); recall requires annotating the whole database per query (p. 467-468). Rank-aware options: NDCG, MAP, MRR (p. 468).
- **Chunking**: split documents by fixed units (chars/words/sentences/paragraphs), recursively, or by tokens of the generator's tokenizer; overlap chunks so boundary information survives ("I left my wife | a note") (p. 474).
- **Query rewriting**: reformulate ambiguous follow-ups ("How about Emily Doe?") into standalone queries before retrieval, via heuristics or another model (p. 476-477).
- **Contextual retrieval**: augment each chunk with metadata, likely questions it answers, or a 50-100 token AI-generated context situating it in the original document (Anthropic's approach) (p. 478-480).
- **Tool inventory**: the set of tools an agent can access — three categories: knowledge augmentation (retrievers, web browsing), capability extension (calculator, code interpreter, translators), and write actions (send email, initiate transfer) (p. 493-497).
- **Function calling**: declare tools (entry point, parameters, docs), control per-query with required/none/auto, model generates tool name + arguments; APIs can guarantee valid function names but not correct parameter values (p. 510-513).
- **Planning granularity**: detailed plans are harder to generate but easier to execute; natural-language plans ("get current date") are robust to tool API renames but need a translator into executable commands — translation is easier than planning, so a weaker model can do it (p. 513-514).
- **Control flow**: sequential, parallel, if-statement, for-loop orderings of plan actions; check which ones an agent framework supports — parallel execution cuts perceived latency (p. 514-516).

## Mental Models
- Think of context construction for foundation models as feature engineering for classical ML — same purpose, giving the model the information to process an input (p. 453).
- Use term-based retrieval as your baseline: fast, cheap, strong out of the box, but little to tune. Move to embedding-based when you can invest in finetuning and need semantic matching — it can be improved over time; term-based can't, much (p. 466).
- Think of an agent's accuracy as compounding per step: 95% per-step accuracy → 60% over 10 steps → 0.6% over 100. This is why agents need stronger models than single-shot use cases (p. 491).
- Think of planning as a search problem with backtracking — you need not just available actions but each action's predicted outcome state; chain-of-thought alone (action sequences without outcome prediction) is insufficient (p. 504).
- Use the frequency-of-use rule for memory placement: needed by all tasks → train/finetune into internal knowledge; rarely needed → long-term memory; immediate task-specific → short-term memory (p. 532).

## Anti-patterns
- **Assuming long context kills RAG**: data grows faster than context windows, models use long context poorly, and every extra token costs money and latency (p. 453-454).
- **Coupled planning and execution**: a model can run a useless 1,000-step plan for hours before you notice; validate plans before executing (p. 500).
- **Verbatim retrieval of conversational follow-ups**: ambiguous queries retrieve garbage; rewrite first — and if identity resolution fails, the rewriter must say so instead of hallucinating a name (p. 477).
- **Embedding away keywords**: embeddings obscure exact strings like error code EADDRNOTAVAIL(99) or product names; add them to chunk metadata or use hybrid search (p. 466, p. 478).
- **FIFO short-term memory eviction**: assumes early messages matter least, but the earliest messages often state the conversation's purpose — "fatally wrong" in those cases (p. 536).
- **Too many tools**: more tools = more capability but harder to use well and bigger tool descriptions eating context; ablate — if removing a tool doesn't drop performance, remove it (p. 522-523).
- **Trusting reflection blindly**: agents can be convinced they succeeded when they haven't (assigned 40 of 50 people to rooms, insists it's done) (p. 528).

## Code Examples
```text
SYSTEM PROMPT
Propose a plan to solve the task. You have access to 5 actions:
  get_today_date()
  fetch_top_products(start_date, end_date, num_products)
  fetch_product_info(product_name)
  generate_query(task_history, tool_output)
  generate_response(query)
The plan must be a sequence of valid actions.

Examples
Task: "Tell me about Fruity Fedora"
Plan: [fetch_product_info, generate_query, generate_response]
Task: "What was the best selling product last week?"
Plan: [fetch_top_products, generate_query, generate_response]

Task: {USER INPUT}
Plan:
```
- **What it demonstrates**: turning a model into a plan generator via prompt engineering — plans as function-name sequences whose parameters are inferred later from prior tool outputs (p. 506-508).

## Reference Tables

**Term-based vs embedding-based retrieval (Table 6-2, p. 470)**

| | Term-based (BM25, Elasticsearch) | Embedding-based |
|---|---|---|
| Querying speed | Much faster | Query embedding + vector search can be slow |
| Performance | Strong out of the box, hard to improve; wrong docs from term ambiguity | Can outperform with finetuning; handles natural, semantic queries |
| Cost | Much cheaper | Embedding, vector storage, and search can be expensive — vector DB spend can hit 1/5 to 1/2 of model API spend (p. 469) |

**Vector search (ANN) algorithm options (p. 464-466)**

| Algorithm | Approach | Notes |
|---|---|---|
| LSH | Hash similar vectors into buckets | Trades accuracy for speed; in FAISS, Annoy |
| HNSW | Multi-layer graph, traverse edges of similar vectors | High accuracy, fast queries; heavy build time/memory (p. 471); in FAISS, Milvus |
| Product Quantization | Decompose vectors into low-dim subvectors | Faster distance computation; key FAISS component |
| IVF | K-means clusters (~100-10K vectors each); search nearest centroids' clusters | With PQ, backbone of FAISS |
| Annoy | Multiple binary trees with random splits | Tree-based; Spotify open source |

Evaluate with ANN-Benchmarks metrics: recall, queries per second, build time, index size (p. 471-472); retrieval systems broadly with BEIR; embeddings with MTEB (p. 469, 472).

**Agent failure modes and what to measure (p. 527-531)**

| Failure class | Examples | Metrics |
|---|---|---|
| Planning: tool use | Invalid tool (bing_search not in inventory); valid tool, invalid parameters; valid tool, incorrect parameter values | % valid plans; plans needed per valid plan; % valid tool calls (p. 528-529) |
| Planning: goal | Wrong destination; over budget; ignored time constraint; false "done" from reflection error | Constraint adherence per (task, tool inventory) tuple |
| Tool | Correct tool, wrong output (bad SQL, wrong caption); translation errors; missing tools for a domain | Test each tool independently; print every call + output (p. 529-530) |
| Efficiency | Valid but wasteful | Steps per task, cost per task, time per action — vs baseline agent or human (p. 530-531) |

## Worked Example
**Kitty Vogue sales-projection agent (p. 490-491).** A RAG-over-tabular-data agent with three actions (response generation, SQL query generation, SQL query execution) receives: "Project the sales revenue for Fruity Fedora over the next three months."

1. Reason: to predict future sales, it first needs the last five years of sales numbers (reasoning is emitted as intermediate response).
2. Invoke SQL query generation for five years of sales numbers.
3. Invoke SQL query execution.
4. Reflect on outputs: numbers are insufficient (missing values) — it also needs past marketing campaign data.
5. Invoke SQL query generation for campaign queries.
6. Invoke SQL query execution.
7. Reason the information now suffices; generate the projection.
8. Reflect that the task is complete.

Note the pattern: parameters can't be predicted upfront — if get_time() returns "2030-09-13", the next call becomes fetch_top_products(start_date='2030-09-07', end_date='2030-09-13', num_products=1) (p. 509). Both action sequences and parameters are model-generated, so both can be hallucinated — always log and inspect parameter values per call (p. 510, 513). The plain-SQL variant of this flow: text-to-SQL → SQL execution → generation (p. 485-486).

## Key Takeaways
1. Start retrieval with BM25/Elasticsearch as a baseline; add embedding-based retrieval and hybrid search only when semantics or finetuning headroom justify the cost (p. 466, 470).
2. Evaluate a RAG system at three levels: retrieval quality (precision/recall), embedding quality, and end-to-end output quality (p. 472).
3. There is no universal best chunk size or overlap — experiment; smaller chunks mean more diverse context but risk losing information and double the indexing/storage cost per halving (p. 474-475).
4. Use retrieval optimization tactics in order of leverage: chunking strategy, reranking, query rewriting, contextual retrieval (p. 474).
5. Decouple planning from execution, validate plans before running them, and put humans in the loop for risky write actions (DB updates, transfers) with per-action automation levels defined (p. 500-502).
6. Reflection is not mandatory for an agent to operate but is necessary for it to succeed — it's cheap to implement (self-critique or a separate evaluator) with big gains, at the cost of tokens and latency (p. 503, 517, 520).
7. Give agents write actions only with trust and security in place — "just as you shouldn't give an intern the authority to delete your production database" (p. 497).

## Connects To
- **Ch 1**: RAG and agents are rung two of ch01's adaptation ladder — context over weights.
- **Ch 2**: structured outputs make function calling reliable; retrieved context is the main hallucination mitigation.
- **Ch 3**: embeddings and their evaluation (MTEB) power semantic retrieval; AI judges validate plans and act as reflection evaluators.
- **Ch 4**: local factual consistency is RAG's key output criterion; agent failure-mode metrics extend ch04's component-level evaluation pipeline.
- **Ch 5**: prompt instructions are the per-application half of context; tools and write actions are the indirect prompt-injection surface ch05 defends against.
- **Ch 7**: the RAG-vs-finetuning decision rule — information failures here, behavioral failures there; RAG first when both (p. 539).
- **Ch 8**: vector-search techniques are reused for semantic deduplication; agent/tool-use traces are prime synthetic-data targets.
- **Ch 9**: every retrieved token costs latency and money — prompt caching and inference-with-reference specifically accelerate retrieval-heavy and multi-turn workloads.
- **Ch 10**: context construction is architecture step 1 and agent patterns step 5; write actions demand guardrails and human approval.
- **Information retrieval / RecSys**: retrieval, vector search, and reranking are century-old IR machinery reused for RAG (p. 457).
- **Reinforcement learning**: FM agents vs RL agents — same environment/action framing, different planner training; Reflexion echoes actor-critic (p. 505).
