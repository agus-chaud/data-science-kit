---
name: ahmad-30-agents
description: Knowledge base from "30 Agents Every AI Engineer Must Build" by Imran Ahmad, PhD (Packt, 2026). Use when applying the book's frameworks — cognitive loop, agent-brain paradigms (reactive/deliberative/hybrid), the 5 levels of agent interaction, the Agentic AI Progression Framework (Manual→Reactive→Tool-using→Planning→Learning), PTCF prompting, RAG pipelines, tool orchestration, multi-agent/chain-of-agents, MCP/A2A protocols, ethical & explainable agents, POMDP reasoning, TDG/Test-Driven Generation, knowledge tracing, particle filter localization, agent societies, the agent development lifecycle — or when building any of the book's 30 named agents, e.g. the Autonomous Decision-Making, Planning, Memory-Augmented, Knowledge Retrieval, Document Intelligence, Chain-of-Agents Orchestrator, Verification and Validation, Code-Generation, Self-Improving, Vision-Language, Healthcare Intelligence, Financial Advisory, Education Intelligence, Collective Intelligence, Embodied Intelligence, or Domain-Transforming Integration agents — or any domain-specific agent pattern (healthcare, financial, education, software, embodied).
allowed-tools: [Read, Grep]
argument-hint: [topic, framework name, or chapter number]
---

# 30 Agents Every AI Engineer Must Build
**Author**: Imran Ahmad, PhD | **Publisher**: Packt (2026) | **Pages**: 542 | **Chapters**: 16 + Epilogue | **Generated**: 2026-05-19

## How to Use This Skill

- **Without arguments** — `/ahmad-30-agents` loads core frameworks for immediate reference
- **With a topic** — `/ahmad-30-agents RAG` or `/ahmad-30-agents tool orchestration` → I find the relevant chapter
- **With chapter** — `/ahmad-30-agents ch06` → I load that specific chapter
- **Browse** — "what chapters do you have?" → full index below

When you ask about a topic not in Core Frameworks, I read the relevant chapter file before answering.

---

## Core Frameworks & Mental Models

### The Cognitive Loop — Universal Agent Foundation
Every agent, regardless of domain, implements this:
```
Perceive → Reason → Plan → [Safety Check] → Execute → Learn
                                 ↑                        |
                                 └────────── Feedback ────┘
```
Use this as your debugging framework. When an agent misbehaves, identify which stage failed.

### Agent Brain — Reactive / Deliberative / Hybrid
The structural choice of HOW an agent reasons — a long-term commitment, not an implementation detail:
- **Reactive**: input → predefined action. Stateless, memoryless, cannot learn. Lowest latency. Use for time-critical, simple mappings (smoke detector, motion light, fast customer-facing routing).
- **Deliberative**: pauses, analyzes the environment, projects outcomes before acting. Needs monitoring, guardrails, and human-escalation fallback. Use where contextual accuracy and compliance matter (travel planning, autonomous navigation, financial planning).
- **Hybrid**: reactive layer handles time-critical events (via event bus / message queue) while a deliberative layer holds goals + state and can override the reactive layer. Bidirectional, shared memory. Use for resilient real-world systems (enterprise automation, healthcare diagnostics; e.g. a delivery robot whose reactive layer stops instantly on an obstacle while its deliberative layer re-plans the route).

This is about reasoning STYLE — not to be confused with the two five-rung ladders below (maturity and operational autonomy).

### The 6 Agent Traits — Evaluation Checklist
Before calling anything an "agent," verify: **Autonomy** (no continuous human guidance), **Persistence** (state across interactions), **Reactivity** (real-time environment response), **Proactiveness** (goal-initiated action), **Adaptability** (learns from experience), **Goal-orientation** (plans under uncertainty). Fewer than 4 → it's a pipeline.

### Agentic AI Progression Framework — Maturity Ladder (Figure 1.14)
Five maturity levels graded across three dimensions: **autonomy, reasoning, adaptability**:
- **Level 0: Manual operations** — non-agentic. Humans initiate, execute, and oversee everything; digital systems are mere tools (analysts hand-building monthly reports).
- **Level 1: Reactive agents** — rule-based automation. Deterministic trigger → preprogrammed action, stateless and context-free (templated email autoresponders, RPA bots, basic voice assistants).
- **Level 2: Tool-using agents** — augmented execution. Parse natural-language instructions, select external tools by context, chain operations; session-scoped context only (PDF-extraction-to-database systems, multi-source report generators).
- **Level 3: Planning agents** — contextual and goal-oriented. Decompose high-level objectives into task sequences, incorporate intermediate feedback, re-plan on obstacles, persistent awareness across extended operations (autonomous travel planners, adaptive project management).
- **Level 4: Learning agents** — adaptive and evolving. Incorporate feedback from past interactions, build personalized models, continuously refine strategies (fraud detection that evolves with attack patterns, autonomous research agents refining hypotheses).

Use this to assess where your system sits and what to invest in next. Don't skip levels.

### Five Levels of Agent Interaction — Operational-Autonomy Axis
Ahmad's SECOND five-rung ladder, ranked by operational autonomy + contextual awareness + decision authority:
1. **Direct LLM** — no autonomy, human-led. Creative generation, FAQ.
2. **Proxy agent** — turns unstructured input into strict structured output (JSON / SQL) for backend systems. Input sanitization against prompt injection; full prompt/response logging. Low autonomy, instruction-based. Use for API parameterization, financial-instruction validation, healthcare triage intake.
3. **Assistant system** — has tool/service access, operates with the user in the approval loop. Medium autonomy, session-scoped context. Use for enterprise assistants, customer-service bots.
4. **Autonomous agent** — long-horizon tasks with partial-autonomy decisions + persistent memory. Needs policy constraints, HITL checkpoints, behavior monitoring. Built on LangGraph / LangChain / CrewAI.
5. **Multi-agent system** — many autonomous agents coordinate via pub/sub, shared memory, or task dispatch; fault tolerance via health checks + task reallocation. Very high autonomy, distributed decisions. Use for self-driving, trading, supply chains.

> ⚠️ **Two different five-rung ladders — do NOT conflate:**
> - **Agentic AI Progression** (Figure 1.14, above) = MATURITY, how evolved the system is: Manual operations (L0) → Reactive (L1) → Tool-using (L2) → Planning (L3) → Learning (L4). Graded on autonomy + reasoning + adaptability.
> - **Five Levels of Interaction** (this one) = OPERATIONAL AUTONOMY, how much it's trusted to act: Direct LLM → Proxy → Assistant → Autonomous → Multi-agent.
> Both have five rungs, both have a low "reactive/direct" rung, and both peak in something called "autonomous"/"multi-agent" — that's the trap. One grades capability maturity of a single system; the other grades decision authority and interaction paradigm. A Level 4 Learning agent can operate as a mere Level 3 Assistant, and a Multi-agent system can be composed of Level 1 Reactive members.

### PTCF — System Prompt Template
**Every** agent system prompt uses this structure:
- **P**ersona: Who the agent is, its role, positive behaviors, hard constraints
- **T**ask: Objective, scope, explicit exclusions
- **C**ontext: Background knowledge, rules, environmental constraints
- **F**ormat: Output structure, required fields, schema

### Three-Tier Memory Architecture
| Type | Storage | Scope | Use For |
|---|---|---|---|
| Working | Context window | Per conversation | Current task |
| Episodic | Vector DB / SQL | Per user, persistent | Personalization, history |
| Semantic | RAG index | Global, domain | Domain knowledge |

### Tool-Using Agent — 4 Components
1. **Reasoning core**: Plans which tools to call and in what sequence
2. **Tool registry**: Catalog with typed schemas (Pydantic) — the contract
3. **Execution engine**: Manages retries, timeouts, error recovery
4. **Tool chest**: Actual implementations, each with: timeout + exception handling + input/output validation

### Chain-of-Agents (CoA) — Multi-Domain Orchestration
Supervisor decomposes → routes to specialists → synthesizes. Use when tasks span domains requiring deep per-domain expertise. Each specialist has its own narrow tool set.

### MCP & A2A — Interoperability Protocols
Two standards that let agents talk to tools and to each other without bespoke integration:
- **MCP (Model Context Protocol)** — universal interface between an agent and tools/APIs/data sources. Three moves: (1) **Capability description** — each tool registers inputs/outputs/constraints in a machine-readable format; (2) **Discovery** — the agent queries the layer to find the right tool for the task; (3) **Invocation** — the agent calls it through the standard protocol, no tool-specific glue. Lets you swap or upgrade tools without touching agent logic.
- **A2A (Agent-to-Agent)** — how agents share intent and coordinate in multi-agent systems. Contract-based (protobuf / JSON schema). Exchanges three things: **state** (context + intermediate results), **role** (functional responsibility in the workflow), **status** (lifecycle: success / failure / readiness). Messaging via Kafka or RabbitMQ; native support in CrewAI and LangGraph.

### Advanced RAG — Production Checklist
Chunking (type-aware) + Metadata (doc_type, date, section) + Hybrid retrieval (vector + BM25) + Re-ranking (cross-encoder) + Structured context assembly. All 5 required for production. Naive embed+search+generate fails at scale.

### Ethical Architecture — The Key Invariant
Ethics is **structural**, not behavioral. The ethical checkpoint goes **between Plan and Execute** — not on the output. Content filters are NOT ethical agents. Value hierarchy: Safety > Ethics > Task.

### POMDP Pattern — Hidden State Reasoning
Used when the critical variable is unobservable (student mastery, patient condition, physical world state). Agent maintains a **belief state** b(s) — probability distribution over possible states — updated with each observation. Actions chosen to perform well across the belief distribution, not just the most likely state.

### Supervised Autonomy — Regulated Domains
In healthcare, finance, and legal: agents analyze and recommend autonomously, but **humans approve before consequential execution**. This is the mandated model for current AI in regulated industries.

### Agent Development Lifecycle (ADL) — Build Process
The end-to-end discipline for shipping an agent, not just architecting one:
1. **Problem space & goals** — model the mental processes the agent must simulate (track user intent, interpret environment, pick strategies, update plans). Map goals into flexible sub-goals; set ethical/technical/operational boundaries.
2. **Architecture & design** — balance modularity, autonomy, extensibility. Define memory strategy, internal comms, external interaction points. Record key calls in **ADRs (Architecture Decision Records)**.
3. **Implementation & integration** — build on LangChain/CrewAI/LangGraph; run local sims or staged deploys. Real constraints surface here (latency, context limits, token cost), forcing engineering trade-offs. Wire agent-behavior tests into CI/CD.
4. **Evaluation & optimization** — metrics: task success rate, avg response time, tool-invocation latency, fallback frequency, escalation rate, user-satisfaction score, robustness under ambiguity.
5. **Governance & lifecycle management** — proactive monitoring, log/compliance auditing, model updates, failure recovery, security patching (LangSmith, Prometheus). Alerts can trigger human review, prompt redesign, fine-tuning, or retraining.

---

## Chapter Index

| # | Title | Key Frameworks |
|---|---|---|
| [ch01](chapters/ch01-foundations.md) | Foundations of Agent Engineering | Cognitive Loop, 6 Traits, Agentic Progression, Agent Development Lifecycle |
| [ch02](chapters/ch02-toolkit.md) | The Agent Engineer's Toolkit | Framework selection, LLM selection, vector DB, observability |
| [ch03](chapters/ch03-prompting.md) | The Art of Agent Prompting | PTCF, cognitive programming, prompt injection defense |
| [ch04](chapters/ch04-deployment.md) | Agent Deployment and Responsible Development | Typology→infra matrix, prompt injection stack, responsible AI |
| [ch05](chapters/ch05-cognitive-architectures.md) | Foundational Cognitive Architectures | Autonomous loop, Tree-of-Thought, 3-tier memory, escalation |
| [ch06](chapters/ch06-retrieval-knowledge.md) | Information Retrieval and Knowledge Agents | Advanced RAG, retrieval failure diagnosis, document intelligence |
| [ch07](chapters/ch07-tool-orchestration.md) | Tool Manipulation and Orchestration Agents | 4-component tool arch, function-calling patterns, CoA, HITL |
| [ch08](chapters/ch08-data-reasoning.md) | Data Analysis and Reasoning Agents | Cognitive feedback loop, verification agent, uncertainty quantification |
| [ch09](chapters/ch09-software-dev.md) | Software Development Agents | TDG (Test-Driven Generation), compliance-driven agents (PCI DSS), self-improving agents |
| [ch10](chapters/ch10-conversational-content.md) | Conversational and Content Creation Agents | Writer-editor loop, brand alignment, dialog management |
| [ch11](chapters/ch11-multimodal.md) | Multi-Modal Perception Agents | Vision-language arch, 4 modality principles, audio, IoT |
| [ch12](chapters/ch12-ethical-explainable.md) | Ethical and Explainable Agents | Ethical checkpoint, value alignment, SHAP/LIME, audit trail |
| [ch13](chapters/ch13-healthcare.md) | Healthcare and Scientific Agents | Layered compliance arch, HIPAA, clinical HITL, scientific research |
| [ch14](chapters/ch14-financial-legal.md) | Financial and Legal Domain Agents | Multi-agent advisory, real-time data pipeline, compliance agent |
| [ch15](chapters/ch15-education-knowledge.md) | Education and Knowledge Agents | POMDP, pedagogical alignment, ZPD, collective intelligence |
| [ch16](chapters/ch16-embodied.md) | Embodied and Physical World Agents | Hierarchical control, particle filter, perception-action loop, safety |
| [ch17](chapters/ch17-epilogue.md) | Epilogue: The Future of Intelligent Agents | Emerging paradigms, agent societies, governance, brain-inspired architectures, ROI |

## Topic Index

- **A2A protocol** → ch01, ch07
- **Agent brain (reactive/deliberative/hybrid)** → ch04, ch05
- **Agent societies / governance / future of agents** → ch17
- **Agent Development Lifecycle (ADL)** → ch01, ch04
- **Audit trail** → ch04, ch12, ch13, ch14
- **AutoGen** → ch02, ch09
- **Brand compliance** → ch10
- **Chain-of-Agents** → ch07, ch09, ch10
- **ChromaDB / FAISS / Pinecone** → ch02
- **Cloud agent platforms (Bedrock / AI Foundry / Vertex AI)** → ch02
- **Cognitive Loop** → ch01, ch05
- **Conflict resolution** → ch07
- **CrewAI** → ch02, ch09
- **Embodied agents** → ch16
- **Episodic memory** → ch05, ch10, ch13
- **Escalation** → ch04, ch05, ch13, ch14
- **Ethical checkpoint** → ch12, ch13, ch14, ch15
- **Financial agents** → ch14
- **Five Levels of Agent Interaction** → ch01
- **Healthcare agents** → ch13
- **HITL** → ch04, ch07, ch13, ch14
- **Impossibility theorem** → ch12
- **LangChain / LangGraph** → ch02, ch07
- **LlamaIndex** → ch02, ch06
- **MCP protocol** → ch01, ch07
- **Multi-agent** → ch07, ch09, ch10, ch14, ch15
- **POMDP** → ch13, ch15, ch16
- **Prompt injection** → ch03, ch04, ch09
- **Proxy agent** → ch01
- **PTCF** → ch03
- **RAG / retrieval** → ch06
- **Safety check** → ch04, ch05, ch12, ch16
- **SHAP / LIME** → ch12
- **Software dev agents** → ch09
- **Tool orchestration** → ch07
- **TDG / Test-Driven Generation** → ch09
- **Tree-of-Thought** → ch03
- **Value alignment** → ch12
- **Vision-Language** → ch11
- **Zero trust** → ch04
- **ZPD** → ch15

## Supporting Files

- [agents-index.md](agents-index.md) — all 30 agents: canonical name → book chapter → skill chapter file → repo folder → simulation notebook, plus book-vs-repo discrepancies. **Read this first whenever a request names an agent** (e.g. "Verification and Validation Agent" → Ch 8 → `chapters/ch08-data-reasoning.md` → `chapter08/`) — it resolves the name to chapter, file, and runnable code in one hop, and flags the names where the repo and the book disagree (the repo's "Security-Hardened Agent" is the book's *Compliance-Driven agents*; the repo's "Recommendation Agent" does not exist in the book).
- [glossary.md](glossary.md) — all key terms with definitions and chapter references
- [patterns.md](patterns.md) — all techniques and design patterns with trade-offs
- [cheatsheet.md](cheatsheet.md) — quick-reference tables, decision matrices, safety checklists

---

## Companion Repo

Local clone: `C:\Users\Dell\Agus\Ai Agents Imran Ahmad\30-Agents-Every-AI-Engineer-Must-Build\`

**Mapping**: repo `chapterNN` ↔ skill `chapters/chNN-*.md` (chapter03 ↔ ch03-prompting.md, ... chapter16 ↔ ch16-embodied.md, chapter17 ↔ ch17-epilogue.md). chapter01/chapter02 hold setup material; agent code starts at chapter03. Per-agent mapping: [agents-index.md](agents-index.md).

**Standard folder structure** (every `chapterNN`):
- 5 provider notebooks per chapter topic: base `chNN_<topic>.ipynb` plus `__RUN_OPENAI_GPT4o`, `__RUN_CLAUDE_Sonnet4`, `__RUN_GEMINI_Flash25`, `__RUN_LOCAL_OLLAMA_DeepSeek_V2_16B`, and `__RUN_NO_KEY_SIMULATION` variants. The `*NO_KEY_SIMULATION*` notebook runs without any API key via a `MockLLM` (deterministic canned responses).
- `USECASE.md` — fictional-company case study the chapter code solves
- `LLM_COMPARISON.md` — cross-provider output comparison
- `TROUBLESHOOTING.md` / `troubleshooting.md` — common failures (casing varies by chapter)
- `AGENTS.md` — agent-facing repo instructions
- Per-provider `requirements-{openai,claude,gemini,ollama}.txt` + shared `requirements.txt`, `LOCAL_LLM_SETUP.md`
- Helper modules vary by chapter (`mock_llm.py`, `resilience.py`, `agent_utils.py`, etc.)

**Errata** (repo `ERRATA.md`): ch5 p.118 — the book prints `cd .../"Chapter 05"`; the real path is lowercase `chapter05` (all folders are `chapterNN`, no spaces).

## Scope & Limits

For hands-on implementation in your codebase, combine with project-specific tools. Code examples in the book use Python 3.10+, LangChain/LangGraph 0.2+, OpenAI API, Anthropic API (Claude 3.5+), ChromaDB/FAISS.
