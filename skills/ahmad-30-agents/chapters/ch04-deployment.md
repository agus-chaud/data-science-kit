# Chapter 4: Agent Deployment and Responsible Development

## Core Idea
Deployment is the deployment phase of the Agent Development Lifecycle (ADL) introduced in Ch1 — the most consequential and risk-sensitive stage (Ch 4, p. ~124). Agents differ from traditional software in being non-deterministic, autonomous, and often stateful, with decision logic emerging from interaction and learning (RLHF, in-context learning via few-shot examples, continual fine-tuning on deployment data) (Ch 4, p. ~124). Consequently they **scale on cognitive load** — task complexity, memory state, reasoning depth, tool dependency — not on request volume, and infrastructure must mirror the agent's cognitive architecture (Ch 4, p. ~125). The chapter's governing claim: infrastructure becomes a **mirror of cognition** and a **scaffold for autonomy**, and typology–infrastructure alignment is a **deployment determinant** (Ch 4, p. ~128). Motivating statistic: 70–80% of AI projects never reach production, and many that do fail within the first year from infrastructure inadequacies, security vulnerabilities, or ethical oversights (Ch 4, p. ~125). Four covered topics: scaling agent systems, handling high-throughput agent interactions, security and privacy, ethical agent development (Ch 4, p. ~125).

## Frameworks Introduced

- **Infrastructure requirements by agent typology** (Figure 4.1) — cognitive architecture dictates deployment strategy (Ch 4, p. ~126):

  | Agent type | Execution mode | Deployment target | Coordination mechanism |
  |---|---|---|---|
  | Reactive | Stateless functions | Serverless/edge | Event trigger (HTTP/SQS/webhook) |
  | Deliberative | Stateful, compute-bound | GPU VMs / cloud containers | Planning DAGs, checkpointing |
  | Hybrid | Context-aware multistage | Microservice clusters | Internal message bus, fallback |
  | Multi-agent | Distributed and autonomous | Kubernetes/mesh + Kafka | Messaging, vector context, roles |

  - **Reactive**: stateless, reflex-driven; minimal CPU, negligible memory footprint; ultra-low-latency use cases — customer service chatbots, simple decision trees, real-time filtering (Ch 4, p. ~126).
  - **Deliberative**: state-rich, goal-directed; extensive modeling, planning, and simulation; substantial CPU **and GPU** for planning-tree generation and multi-step reasoning chains; needs long-context tracking, persistent knowledge graphs, and intermediate reasoning states across extended sessions. Named tooling: LangChain `ConversationBufferMemory` / `ConversationSummaryMemory` for short-to-medium context, Pinecone for scalable long-context vector retrieval (Ch 4, p. ~127).
  - **Hybrid**: the most complex architectural challenge — dual-mode, switching between reflexive and deliberative behavior based on task semantics and environmental conditions; must simultaneously serve low-latency routine traffic and heavyweight planning (Ch 4, p. ~127).
  - **Multi-agent**: distributed computing challenges — messaging infrastructure, coordination protocols, state synchronization; Kubernetes plus Kafka, RabbitMQ, or cloud-native pub/sub; supports both peer-to-peer and centralized coordination. Observability is *particularly* critical here because behavior emerges from agent interactions rather than predetermined logic flows. Use cases: supply chain agents negotiating contracts across vendor ecosystems, healthcare agents coordinating patient data across clinics and insurers (Ch 4, p. ~127).

- **Performance optimization triad** — a performant agent balances three competing factors (Ch 4, p. ~128): **cost efficiency** (every action and inference must provide value), **high throughput** (large concurrent volumes without degradation), **resilience and latency** (stability when downstream services fail or load spikes).

- **Cost optimization framework** (Figure 4.2) — five interconnected strategies; improvements in one enhance the others, and neglecting any undermines the whole (Ch 4, p. ~129):
  1. **Model selection and routing** (foundational): lightweight vs. heavyweight models — GPT-3.5 for FAQ lookups and greetings, GPT-4 for complex problem-solving; **confidence-based escalation** routes upward when the small model's confidence falls below a threshold (Ch 4, p. ~129).
  2. **Tiered architecture and routing**: Tier 1 = lightweight classifiers or rule-based systems (intent recognition, spam filtering, basic validation); Tier 2 = intermediate models for most conversational interactions; Tier 3 = high-accuracy, expensive models invoked selectively for creative or nuanced tasks. Routing driven by classification models, decision trees, or metadata-driven policies (user priority, task urgency, estimated cost) (Ch 4, p. ~129–131).
  3. **Response caching and output reuse**: API result caching with **TTL** metadata; embedding caches for frequently accessed knowledge bases; LLM response caches for common completions (Ch 4, p. ~130).
  4. **Cost-aware routing and budget enforcement**: classify tasks with **SLA** tagging and cost ceilings; budget-aware middleware monitors spend in real time and applies **graceful degradation** — routing to cached responses, defaulting to lower-cost models, or queuing non-urgent tasks (Ch 4, p. ~130).
  5. **Monitoring and iterative optimization**: observability pipelines with token cost dashboards tracking usage by model, route, and agent type; anomaly detection for cost spikes and excessive prompt lengths. Named tooling: **Grafana** and **OpenAI's billing API**. Best practice — tag every model call with `task_id`, `agent_type`, and `cost_tier` for fine-grained attribution (Ch 4, p. ~132).
  - Two further levers listed alongside these: **prompt compression and token optimization** (semantic compression of interaction history, structured prompt templates, context-window pruning of low-entropy content) and **conditional execution and token gating** (route deterministic tasks to templates, scripts, or lookup tables with no LLM involvement) (Ch 4, p. ~132).

- **Cost vectors** — the five main sources of operational expense (Ch 4, p. ~130–131): token consumption (prompt volume, completion length, context tokens, verbose internal monologues, exponential message overhead in multi-agent systems); inference duration (GPU/accelerator time); tool and API calls (unpredictable, driven by agent reasoning rather than user input volume); memory and storage (persistent embeddings, conversation histories, vector DBs with expensive similarity operations); inter-agent messaging (potentially **quadratic** growth in message volume). Guiding principle: efficient systems prioritize **value per unit of compute**, since under-utilizing intelligence is often the greatest inefficiency (Ch 4, p. ~131).

- **Resilience patterns for high-throughput agent systems** (Table 4.1) (Ch 4, p. ~133):

  | Pattern | Purpose | Example tools |
  |---|---|---|
  | Circuit breakers | Block failing services to avoid cascade | Hystrix, Istio, Sentinel |
  | Bulkheads | Isolate failure domains per agent type | Kubernetes namespaces, Helm charts |
  | Timeout + retry wrappers | Fail fast and reroute intelligently | Tenacity (Python), Resilience4j |
  | Failover models | Fall back to cached or distilled responses | Prompt chaining, decision DAGs |

  - Book's worked example: a customer support agent handling **500 concurrent sessions** with a **2-second end-to-end response SLA**. At that scale one slow reasoning chain or unresponsive tool call cascades into SLA breaches. Circuit breakers can reduce **P99 latency by 40–60%** during failure scenarios by failing fast or switching to fallback responses; bulkhead isolation keeps a degraded retrieval service from affecting unrelated workflows (Ch 4, p. ~132).
  - High-throughput agent systems are characterized by **complexity-per-unit-of-traffic**, measurable as average LLM calls per user request, reasoning-chain depth, external tool invocation count, or inter-agent messages per interaction (Ch 4, p. ~132–133).
  - **Decision DAG vs. retry loop vs. prompt chaining** (the chapter's explicit note): a retry loop re-executes the same node with no branching; prompt chaining is a fixed linear sequence; in a decision DAG each node evaluates a condition and the execution path is determined by that outcome — on failure it routes to a *specialist node* rather than re-attempting the same prompt, and it can diverge across multiple downstream paths (Ch 4, p. ~133).

- **Modular cognition via microservices** (Table 4.2) — decomposition of agent functionality into independently deployable services, forming a **composable agent infrastructure** enabling targeted scaling (Ch 4, p. ~134):

  | Service | Responsibility |
  |---|---|
  | Planner | Translates intent into executable steps |
  | Retriever | Performs semantic or keyword-based search |
  | Memory Store | Manages embeddings, chat logs, and context graphs |
  | Execution Engine | Interfaces with APIs, tools, or external systems |
  | Response Synthesizer | Composes and formats final agent output |

  - Communication via HTTP, gRPC, or message queues. Concrete deployment: Planner and Execution Engine as separate FastAPI or gRPC services in distinct Kubernetes Pods with independent resource quotas; Memory Store connects to Pinecone or Weaviate; Execution Engine holds the registry of callable APIs behind an authorization check so no other service invokes tools directly. Effect: a surge in tool-call volume does not starve the reasoning pipeline of CPU, and a memory retrieval failure trips a circuit breaker only inside that service (Ch 4, p. ~134).
  - The chapter's code example wraps a tool call with `tenacity` retry + exponential backoff behind a manual circuit breaker (`FAILURE_THRESHOLD = 3`). Stated gotcha: in Kubernetes, `CIRCUIT_OPEN` and `FAILURE_COUNT` must live in a **shared Redis key**, not module-level globals, so all Pod replicas observe the same circuit state (Ch 4, p. ~135).

- **Deployment architecture patterns** — the chapter's four-part architectural backbone (Ch 4, p. ~137):
  - **Portable execution with containerization**: Docker for packaging, registries for versioning and rollback; multi-stage builds to minimize image size and protect build-time secrets; Docker BuildKit for advanced caching and secret handling; OCI-compliant runtimes **containerd** and **Podman** where Docker daemon dependencies are undesirable (Ch 4, p. ~135).
  - **Orchestration for lifecycle management**: Kubernetes as core engine, Helm charts for templated deployment, ArgoCD/Flux for GitOps. Deployments for long-running agents, Jobs for ephemeral inference tasks, Custom Operators for memory checkpoints or multi-step workflows. Autoscaling driven by inference latency, memory load, or tool error rate (Ch 4, p. ~135).
  - **Asynchronous messaging and event-driven agents**: Kafka for durable high-throughput streaming, RabbitMQ/NATS for lightweight low-latency queues, Cloud Pub/Sub for managed pub/sub. Patterns: **fan-out**, **dead-letter queues**, **schema contracts** (Ch 4, p. ~136).
  - **Hybrid orchestration: stateless routing and stateful agents** — separate orchestration (stateless: routing, retries, policies) from reasoning (stateful: context persisted in external stores such as Pinecone, Redis, ChromaDB). Supports horizontal scaling of orchestrators and fault-tolerant agent execution; long-lived state checkpointed to blob storage or document databases for auditability and replay (Ch 4, p. ~136–137).

- **Operational procedures: rollback, A/B testing, migration** (Ch 4, p. ~136):
  - **Rollback** is harder than standard service rollback because agent state spans four substrates — model weights, tool configurations, memory contents, conversation history — each versioned independently. Kubernetes rolling updates revert the serving container; a *separate migration script* restores the memory store to its pre-deployment snapshot; tool API versions must be explicitly pinned so a rolled-back orchestrator does not call a newer tool schema. Rollback readiness validated in staging before every production deployment.
  - **A/B testing**: traffic-splitting across versions (e.g., a fine-tuned variant against baseline on 10% of production sessions). Must instrument *beyond latency* — task completion rate, tool call frequency, escalation rate, user satisfaction signals — because LLM behavioral regressions often do not manifest as latency or error anomalies. Recommended pattern: canary deployments with automatic rollback triggers on behavioral metric thresholds.
  - **Migration and enterprise integration**: four transition points — in-process → remote tool calls; in-memory → persistent conversation storage; integration with enterprise identity providers (**OAuth 2.0, SAML**) for user-scoped tool authorization; connection to CRMs/ERPs/data warehouses via authenticated API gateways rather than direct database access. **Blue-green deployment** minimizes migration blast radius.
  - **Event schema evolution**: version event contracts with semantic versioning (e.g., `event.v2.user_action`) so consumers detect breaking changes; categorize errors at processing time — transient failures retry with backoff, permanent failures (schema violations, unroutable events) go to a dead-letter queue (Ch 4, p. ~136).

- **Federated architectures across organizations** — agents operating across organizational boundaries with policy-governed collaboration (Ch 4, p. ~137). Three key considerations: **Federated Memory Graphs** (distributed knowledge with scoped permissions), **authentication protocols** (OAuth2, JWT, mTLS for cross-organizational trust), and **Policy Enforcement Points (PEPs)** for runtime rule evaluation. Privacy-preserving techniques: encrypted queries, access filtering, differential privacy. Use cases: supply chain agents negotiating contracts across vendor ecosystems, healthcare agents coordinating patient data, public sector agents handling privacy-restricted citizen data.

- **Threat taxonomy by attack-surface layer** (Table 4.3a) — organizing threats by layer gives a decision-relevant framework for prioritizing defenses (Ch 4, p. ~137):

  | Layer | Threat types | Primary mitigation |
  |---|---|---|
  | Input-level | Prompt injection, adversarial inputs, data poisoning | Input validation; structured prompt schemas; token sanitization |
  | Execution-level | Tool misuse, tool hijacking, identity spoofing, model extraction | Tool gating; least-privilege access; continuous action verification |
  | Memory-level | Memory recall leakage, context poisoning, data leakage | Memory governance; scoped session contexts; PII filtering on memory writes |

  Table 4.3b assigns risk levels: **high** — prompt injection, data leakage, identity spoofing, adversarial inputs, tool hijacking; **moderate** — tool misuse, model extraction; **medium** — indirect prompting (hidden commands embedded in documents or web content), memory recall leakage (Ch 4, p. ~138).

- **Zero trust adapted to AI agents** (Table 4.4) — "trust nothing by default" extended to reasoning processes, contextual memory, and dynamic action selection (Ch 4, p. ~138–139):

  | Principle | Agent-centric implementation |
  |---|---|
  | Least privilege | Limit access to tools, data, and APIs on a **per-task** basis |
  | Continuous verification | Authenticate and authorize actions **per invocation, not per session** |
  | Micro-segmentation | Isolate agent capabilities by namespace, context, or identity |
  | Behavior monitoring | Telemetry and anomaly detection to flag outlier decisions |
  | Immutable infrastructure | Deploy agents in hardened, read-only containers |

  Behavioral drift or unexplained reasoning deviations should automatically trigger containment or escalation (Ch 4, p. ~139).

- **Security incident preparedness** — four measures (Ch 4, p. ~139): **agent-specific runbooks** (procedures for rolling back memory, disabling tools, notifying users); **anomaly detection** (deviations in prompt length, token usage, tool selection, path divergence); **audit trail retention** (all inputs, outputs, and tool interactions in encrypted, append-only systems for **90–180 days**); **red team exercises** (simulated prompt exploits, impersonation attacks, tool hijacking). Named LLM-agent observability tooling: **LangSmith** (prompt and chain tracing with execution replay), **PromptFoo** (prompt regression testing and adversarial evaluation), **OpenLLMetry** (OpenTelemetry-compatible instrumentation for LLM call metrics and spans) (Ch 4, p. ~139).

- **Defense-in-depth architecture** — five layered controls across cognitive and operational boundaries (Ch 4, p. ~140):
  1. **Input validation**: strip malicious tokens, enforce structured prompts, isolate user input from system commands.
  2. **Prompt schema enforcement**: typed input/output definitions to constrain reasoning boundaries.
  3. **Memory governance**: vet updates to persistent memory, constrain memory scope by session or role.
  4. **Tool gating**: whitelist allowed tools, enforce parameter constraints at runtime.
  5. **Interface hardening**: rate-limiting, authentication, sandboxing on all externally exposed endpoints.
  Complemented by behavioral observability platforms — LangSmith, OpenTelemetry, or custom telemetry stacks — for real-time monitoring of agent reasoning and execution (Ch 4, p. ~140).

- **Ethical agent development — four interconnected dimensions** (Figure 4.3); weakness in one compromises the whole framework, so they must be implemented as integrated components of a single ethical architecture (Ch 4, p. ~140–141):
  1. **Transparency and explainability** — foundational; prerequisite for evaluating fairness, ensuring accountability, and maintaining compliance.
  2. **Fairness and bias mitigation** — builds on transparency; verifies equitable treatment across individuals and groups.
  3. **Accountability and oversight** — the governance layer, spanning human oversight and automated monitoring.
  4. **Regulatory compliance** — the external legal framework, integrating with all other dimensions.

## Key Concepts

- **Cognitive load, not request volume, is the scaling axis**: agents scale on task complexity, memory state, reasoning depth, and tool dependency. Retrofitting infrastructure for a misaligned architecture is expensive and difficult (Ch 4, p. ~125).
- **Production readiness stack**: modularize the codebase (microservices), orchestrate workflows (**Temporal** or **Prefect**), harden the runtime (redundancy, circuit breakers), give agents version-controlled identities and explicit tool affordances with validation, and expose introspection via observability frameworks (**OpenTelemetry**) (Ch 4, p. ~124–125). Cloud-native foundation: Docker containerization, Kubernetes orchestration, event-driven integration via Kafka or RabbitMQ, with latency, security, and cost as the balanced architectural properties (Ch 4, p. ~125).
- **Transparency beyond feature importance**: modern explainability offers contextual narratives through multi-modal strategies — visual attention maps, natural-language summaries, interactive tools. Radiology example: highlight suspicious regions *and* explain findings in medical terms, reference similar cases, quantify confidence. Explanations must be tailored per audience (researchers, clinicians, end users). **Longitudinal transparency** = persistently storing complete reasoning trails (model version, training data, environmental context) for future audits (Ch 4, p. ~141).
- **Algorithmic fairness vs. deployment-context fairness** — two distinct problems requiring separate treatment (Ch 4, p. ~142). Algorithmic fairness concerns bias in model predictions relative to protected attributes. Deployment-context fairness concerns bias introduced by *operationalization*: which populations have access, what data routing determines whose queries reach which model, which feedback loops are active, what downstream systems act on outputs. **A model with zero algorithmic bias can still produce unfair outcomes** under restricted access, asymmetric feedback collection, or differential SLA enforcement across user populations.
- **Bias mitigation across the lifecycle**: pre-processing (data augmentation, synthetic data), model development (fairness constraints in optimization objectives), post-processing (adjusting outputs). Advanced frameworks must navigate conflicting fairness notions — **demographic parity vs. equalized opportunity**. Measurement extends beyond statistics to **dignity** and **empowerment** (Ch 4, p. ~142).
- **Audit trails for accountability**: immutable data structures and cryptographic signatures logging agent decisions, environments, and interactions — the technical foundation of accountability (Ch 4, p. ~142).
- **Risk-based escalation for human oversight**: agents defer to human reviewers when confidence is low, populations are vulnerable, or consequences are severe. Oversight design must account for *human* cognitive biases by providing contextual information and highlighting uncertainty (Ch 4, p. ~142).
- **Anticipatory compliance design**: the regulatory environment is evolving rapidly; design for emerging frameworks rather than retrofitting later. Named regimes — **GDPR** and **CCPA** (consent, data access/deletion rights, privacy-by-design), the **EU AI Act** (risk categorization, stringent documentation/testing/human-oversight requirements for high-risk applications), documentation artifacts (**model cards, datasheets**), and **geofencing / data residency controls** for cross-border data flows (Ch 4, p. ~142).
- **NIST AI Risk Management Framework (AI RMF 1.0)** — voluntary, standards-based, four core functions: **Govern, Map, Measure, Manage**. Relevant to agents because it covers organizational accountability, third-party dependencies, and deployment context, not only model-level risk (Ch 4, p. ~143).
- **Executive Order 14110** (Safe, Secure, and Trustworthy Development and Use of AI) — federal reporting requirements for frontier models, mandated red-teaming and safety evaluations before high-risk deployments; increasingly a baseline obligation for agents in regulated sectors or federal infrastructure (Ch 4, p. ~143).
- **Ethics vs. performance is a false dichotomy**: ethically designed systems achieve superior long-term performance, reduced risk, and higher adoption. Transparent systems are more debuggable, fair systems avoid legal costs, accountable systems provide better feedback. Ethical investment enables deployment in regulated markets and attracts talent (Ch 4, p. ~143).
- **Ethically aligned architecture layers**: data architecture (governance policies, quality monitoring, bias detection, privacy-preserving techniques), model architecture (ensembles reduce bias, uncertainty quantification communicates limitations, modularity enables targeted intervention), interface design (clear disclosure of AI involvement, communicated uncertainty, user feedback and control), deployment and monitoring (real-time ethical metrics, stakeholder alerting, iterative refinement) (Ch 4, p. ~143). Reference frameworks: **IEEE Ethically Aligned Design** and **Google's Responsible AI MLOps** practices (Ch 4, p. ~144).
- **Toolchain quick reference** (Table 4.4, second occurrence) — every tool cited in the chapter with its deployment role; prerequisites Python 3.9+ and Docker: Docker, Kubernetes, Temporal, Prefect, OpenTelemetry, Apache Kafka, ArgoCD, Pinecone, Grafana, tenacity, Istio (Ch 4, p. ~144–145).

## Anti-patterns

- **Retrofitting infrastructure onto a misaligned agent architecture** — explicitly called out as expensive and challenging; alignment must be designed in from the start (Ch 4, p. ~125).
- **Perimeter-only security / retrofitted security**: "Security and privacy in AI agents cannot be retrofitted; they must be architected as foundational design principles." Every prompt parsed, tool invoked, and memory write is a risk surface if ungoverned (Ch 4, p. ~140).
- **Treating every prompt/tool call/memory op as trusted**: attack vectors manifest through natural interactions and are hard to detect with traditional static analysis, so each must be treated as a potential compromise entry point (Ch 4, p. ~138).
- **Synchronous communication in distributed agent ecosystems** — introduces fragility; event-driven patterns decouple components for concurrency, resilience, and contextual awareness (Ch 4, p. ~136).
- **A/B testing agents on latency and error rate alone** — LLM behavioral regressions often produce no latency or error anomaly; task completion rate, tool call frequency, escalation rate, and satisfaction signals are the required instruments (Ch 4, p. ~136).
- **Rolling back only the serving container** — agent state spans model weights, tool configs, memory contents, and conversation history; an unpinned tool API version leaves a rolled-back orchestrator calling a schema it no longer matches (Ch 4, p. ~136).
- **Circuit-breaker state in module-level globals under Kubernetes** — replicas then disagree about circuit state; use a shared Redis key (Ch 4, p. ~135).
- **Unversioned event contracts** — without semantic versioning, consumers cannot detect breaking changes and events fail silently, corrupting agent state undetected (Ch 4, p. ~136).
- **Ethics as afterthought** — moral considerations must be embedded through the architecture (data, model, interface, deployment layers), not bolted on (Ch 4, p. ~143).

## Key Takeaways

1. Match infrastructure to agent typology — reactive/serverless, deliberative/GPU + checkpointing, hybrid/microservices, multi-agent/Kubernetes + Kafka. Alignment is a deployment determinant (Ch 4, p. ~126–128).
2. Agent costs scale non-linearly with task complexity; control them with the five interlocking strategies (model routing, tiered architecture, caching, cost-aware routing with budget enforcement, monitoring) and tag every model call with `task_id`/`agent_type`/`cost_tier` (Ch 4, p. ~129–132).
3. Resilience patterns are SLA mechanisms, not just defense: circuit breakers can cut P99 latency 40–60% during failures; bulkheads contain blast radius (Ch 4, p. ~132–133).
4. Separate stateless orchestration from stateful reasoning, and checkpoint long-lived state to blob or document storage for auditability and replay (Ch 4, p. ~136–137).
5. Extend zero trust to the behavioral domain — per-task privilege, **per-invocation** authorization, capability micro-segmentation, behavior monitoring, immutable read-only containers (Ch 4, p. ~139).
6. Defense-in-depth for agents = input validation + prompt schema enforcement + memory governance + tool gating + interface hardening (Ch 4, p. ~140).
7. Transparency, fairness, accountability, and regulatory compliance are one interconnected system; weakness in any one compromises the framework (Ch 4, p. ~140–141).
8. Fairness has two dimensions — algorithmic and deployment-context — and a model clean on the first can still be unfair on the second (Ch 4, p. ~142).
9. Anchor governance in NIST AI RMF 1.0 (Govern/Map/Measure/Manage) and EO 14110; design anticipatorily for GDPR, CCPA, and the EU AI Act rather than retrofitting (Ch 4, p. ~142–143).

## Connects To

- **Ch1**: extends the Agent Development Lifecycle (ADL), detailing its deployment phase (Ch 4, p. ~124)
- **Ch3**: backward bridge — "In Chapter 3, we explored the art of agent prompting"; this chapter operationalizes those prompted agents in production (Ch 4, p. ~146)
- **Ch5**: forward reference — "Chapter 5 will present the cognitive foundational architecture" of the core agents that form the building blocks of all intelligent systems (Ch 4, p. ~146)

## Companion Code
Repo: `30-Agents-Every-AI-Engineer-Must-Build/chapter04/`
- Runs without API key: `ch04_agent_deployment__RUN_NO_KEY_SIMULATION.ipynb` (MockLLM)
- Provider variants: OpenAI GPT-4o / Claude Sonnet 4 / Gemini Flash 2.5 / Ollama DeepSeek local
- Key modules: `agent_utils.py`, `mock_llm.py`
- Context: `USECASE.md`, `LLM_COMPARISON.md`, `TROUBLESHOOTING.md`, `LOCAL_LLM_SETUP.md`, `AGENTS.md`
