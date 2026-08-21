# Chapter 2: The Agent Engineer's Toolkit

## Core Idea
Framework selection is a strategic architectural decision, not a tooling preference. This chapter maps the ecosystem — LangChain, LangGraph, LlamaIndex, CrewAI, AutoGen — and provides a decision matrix for LLM selection, vector database choice, and observability setup.

## Frameworks Introduced

- **Framework Selection Matrix**: Choose based on 4 axes: Control (how much orchestration logic you own), Abstraction (how much the framework hides), Production-readiness (stability, observability, scaling), and Ecosystem (integrations, community). (Ch 2, p. ~69)
  - LangChain/LangGraph: High control, medium abstraction, production-ready → use for custom agent workflows. LangChain is the elder statesman of agent frameworks (70k+ GitHub stars); its computational graph model is a study in modularity. (Ch 2, p. ~70)
  - CrewAI: Low control, high abstraction, good for role-based multi-agent → use for rapid prototyping teams
  - AutoGen: Medium control, conversation-centric, research-grade → use for multi-agent dialogue experiments
  - LlamaIndex: Optimized for RAG/retrieval, excellent data connectors → use when knowledge retrieval is the core

- **LLM Selection Criteria**: Capability tier × Cost tier × Latency requirements × Context window needs, plus the open-weight vs. closed-API axis — open weights give full local deployment and fine-tuning control; closed APIs trade that control for managed operation, with long-term cost implications either way. Never pick a model without benchmarking on YOUR task, not generic leaderboards. (Ch 2, p. ~79)
  - GPT-4o: High capability, high cost → reasoning-intensive tasks with budget
  - Claude 3.5+: Strong reasoning + long context → document-heavy agents
  - GPT-4o-mini / Claude Haiku: Low cost, acceptable quality → high-throughput, simple classification/extraction

- **Vector DB Selection**: Match to scale + query pattern. Retrieval works by embedding the query into the same high-dimensional space as the corpus (768 / 1,024 / 1,536 dims depending on model), then finding stored vectors pointing in similar directions by cosine similarity or dot product. (Ch 2, p. ~81)
  - ChromaDB: Local dev, small-medium collections, zero ops overhead
  - FAISS: In-memory, maximum query speed, no persistence — use for embeddings you rebuild
  - Pinecone/Weaviate: Cloud, managed, high scale → production RAG

## Cloud-Native Agent Platforms

Managed alternatives to OSS frameworks: the three clouds ship multi-agent collaboration, RAG, memory, and guardrails as services. Dominant production pattern is **hybrid** — managed cloud for infrastructure + LLM access, OSS framework for agent logic and custom tools.

- **AWS — Amazon Bedrock Agents** (Ch 2, p. ~86): many FMs (Titan, Claude, Llama, Cohere...) behind one API. Supervisor-orchestrated multi-agent (centralized, not peer-to-peer), managed RAG via Knowledge Bases (incl. NL-to-SQL), API "Action Groups", memory, code interpretation, Guardrails (predefined moderation only — no custom policies). SageMaker for custom/open models. OSS: `langchain-aws`, Strands Agents, MCP. Deploy: Lambda+API Gateway or ECS/EKS.
- **Azure — AI Foundry Agent Service** (Ch 2, p. ~88): an "Agent Factory" — Azure OpenAI models + catalog (Llama, Mistral, Cohere); tools via Logic Apps/Functions/OpenAPI reaching Bing/SharePoint/AI Search; "connected agents" with built-in agent-to-agent messaging; Entra identity, RBAC, network isolation; observability via Application Insights. OSS: Semantic Kernel (deepest), AutoGen, LangChain. Deploy: Functions/Container Apps or AKS.
- **Google Cloud — Vertex AI Agent Builder / Agentspace** (Ch 2, p. ~89): Vertex AI + Model Garden; Agentspace (enterprise search + agent hub, private preview); ADK (open source, framework-agnostic, powers Agentspace); Vertex AI Agent Engine (fully managed, full LangChain/LangGraph integration; AutoGen/LlamaIndex/CrewAI via templates); pushes A2A + MCP open standards against lock-in. Deploy: Cloud Run or GKE.
- **Selection**: AWS if AWS-invested + model variety + large-scale multi-agent; Azure if OpenAI models are mandatory or the org is Microsoft-centric (governance/compliance); Google Cloud for cost-efficient scaling + open standards. Pure OSS when control/portability dominate; managed when time-to-production, guardrails, and scaling matter more.

## Key Concepts

- **Observability stack**: Traces (what did the agent do), Metrics (how fast, how often), Logs (what was the input/output). The book names Prometheus, Grafana, and LangSmith for tracking agent state, action success rates, latency, and error events. Non-negotiable for production. (Ch 2; Ch 1, p. ~40)
- **Fine-tuning vs. prompt engineering**: Fine-tuning is expensive (data, compute, deployment), prompt engineering is free and faster to iterate. Default to prompting; fine-tune only when prompting ceiling is hit.
- **Embedding models**: The bridge between text and vector space. OpenAI text-embedding-3-large outperforms older models significantly. Mismatch between embedding model at index time and query time breaks retrieval silently.
- **Token budget management**: Every agent call has a token cost. Design token budgets upfront — context window management is the #1 source of production surprises.

## Anti-patterns

- **Framework lock-in without evaluation**: Picking LangChain because "everyone uses it" without benchmarking alternatives for your use case wastes months of refactoring later.
- **Ignoring observability until production**: You cannot debug what you cannot observe. Wire LangSmith or Langfuse on day 1, not after launch.
- **Mixing embedding models**: Embedding a knowledge base with model A then querying with model B produces garbage retrieval. Document and enforce embedding model versioning.
- **Over-engineering for scale on day 1**: Start with ChromaDB + SQLite, scale to Pinecone/Postgres when you hit real limits.

## Key Takeaways

1. Use the Framework Selection Matrix before writing a line of code — the wrong framework costs weeks.
2. LLM selection must be task-specific. Benchmark on your eval set, not external leaderboards.
3. Observability (LangSmith/Langfuse) must be installed before the first production call.
4. Embedding model consistency across index and query is a hard requirement, not a suggestion.
5. Token budget management is architectural — design it in from day 1.

## Connects To

- **Ch1** ×5: the chapter's only declared links, all backward — the cognitive loop this toolkit implements (Ch 2, p. ~69); LangGraph conditional routing as the loop's execution substrate (Ch 2, p. ~73); the cognition core of the agent (Ch 2, p. ~78); multi-agent interaction and the Five Levels ladder (Ch 2, p. ~79); MCP and A2A protocols (Ch 2, p. ~80)
- **Ch3** `[pos]`: declared by position — the chapter hands off to advanced prompt engineering as the next step after tooling (Ch 2, p. ~91)
- **Ch6**: LlamaIndex and the vector-DB/embedding stack are the retrieval toolkit that Ch6's four-stage RAG architecture assumes (inferred — not stated in Ch2)
- **Ch7**: LangGraph is the orchestration substrate behind Ch7's chain-of-agents and stateful agentic workflows (inferred — not stated in Ch2)
- **Ch9**: CrewAI/AutoGen are the frameworks behind Ch9's software-development multi-agent team (inferred — not stated in Ch2)

> **Attribution conflict inside the book — MCP/A2A**: Ch2 (p. ~80) credits **Chapter 1** with introducing MCP and A2A, and that is correct — the actual introduction is **Ch1, p. ~47** (MCP standardizes tool discovery/invocation; A2A defines inter-agent message passing). Ch7 (p. ~219) contradicts this by saying they were "introduced in Chapter 6". Route MCP/A2A definitions to Ch1 p. ~47; Ch6/Ch13 are application sites, Ch7 is protocol mechanics.

## Companion Code
Repo: `30-Agents-Every-AI-Engineer-Must-Build/chapter02/`
- Runs without API key: `ch02_agent_toolkit__RUN_NO_KEY_SIMULATION.ipynb` (MockLLM)
- Provider variants: OpenAI GPT-4o / Claude Sonnet 4 / Gemini Flash 2.5 / Ollama DeepSeek local (`ch02_agent_toolkit__RUN_*.ipynb`)
- Key modules: `mock_llm_layer.py`
- Context: `USECASE.md`, `LLM_COMPARISON.md`, `LOCAL_LLM_SETUP.md`
