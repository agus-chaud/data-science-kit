# Patterns & Techniques — 30 Agents Every AI Engineer Must Build

**Index of reusable cross-chapter patterns.** This file points; it does not re-derive. Depth, case studies, worked
numbers, and domain specifics live in `chapters/`. If a "pattern" is really *the agent from chapter N*, it is not here.

Citations use the **EPUB anchor number** (`id="page_NNN"`), which runs **+31 against the printed page**. Do not convert.
Code pointers are verified to exist in `30-Agents-Every-AI-Engineer-Must-Build/`.

| # | Pattern | Ch |
|---|---|---|
| 1 | PTCF system prompt | 3 |
| 2 | Reasoning and planning strategy selection | 3, 5 |
| 3 | Autonomous cognitive loop + escalation gate | 5 |
| 4 | Three-tier memory | 5, 10 |
| 5 | Four-stage RAG with parallel provenance | 6 |
| 6 | Chunking, retrieval tuning, failure diagnosis | 6, 2 |
| 7 | Authority-weighted retrieval + citation gate | 14 |
| 8 | Tool-using agent — registry, schemas, funnel | 7, 8 |
| 9 | Layered failure recovery | 7, 4 |
| 10 | Chain-of-agents orchestration | 7, 14 |
| 11 | Bounded multi-agent deliberation | 15 |
| 12 | Defense-in-depth — five controls | 4 |
| 13 | Safety and ethics upstream of generation | 12, 10 |
| 14 | Verification & Validation meta-reasoning layer | 8, 11 |
| 15 | Calibrated confidence and confidence routing | 13, 6 |
| 16 | Gates in the control flow — validation and human approval | 14, 9, 10, 7 |
| 17 | Regulated-domain architecture — isolation, provenance, audit | 13 |
| 18 | Infrastructure matched to agent typology | 4 |
| 19 | Cost and latency tiering | 4, 13, 15 |
| 20 | Test-Driven Generation with bounded refinement | 9 |
| 21 | Asymmetric control — four-layer hierarchy, admissible envelope | 16 |
| 22 | Belief state over hidden variables (POMDP) | 15, 16 |
| 23 | Vision-Language triad | 11 |

---

## A. Prompting and reasoning

### 1. PTCF system prompt (Ch 3, pp. ~98–103)
**When**: every agent system prompt. Persona, Task, Context, Format — the agent's cognitive contract. Alternative named: CRISPE.
**How**: persona (role + tone) → task (mission + explicit "must not") → context (environment, audience, constraints, conflict-resolution rule) → format (structure, length cap, uncertainty fallback). Every advanced Ch 3 technique is an **elaboration of one PTCF component, not an addition** (Table 3.1, p. ~107). Few-shot belongs in **context** — 2–5 exemplars, up to 10 (pp. ~108–111). Persona is itself a three-layer stack: system prompt → few-shot anchoring → dynamic modulation (Ch 10, pp. ~316–317). Iterate with A/B plus a regression suite, isolating the responsible component before revising (pp. ~121–122).
**Trade-off**: consumes a fixed slice of context on *every* call (3,000 of 4,096 tokens in the book's illustration, p. ~96).
→ `chapters/ch03-prompting.md` · `chapter03/utils.py`

### 2. Reasoning and planning strategy selection (Ch 3, pp. ~113–116; Ch 5, pp. ~162–165)
**When**: choosing how the agent reasons or plans — by *problem structure*, not prestige (Table 3.3).
**How**: **CoT — "the methodical analyst"**, linear step-by-step, fits arithmetic and summarization; specified in PTCF **format**. **ToT — "the virtual strategy team"**, three stages: **decomposition into virtual experts → simulated discussion with pruning → synthesis**; specified in PTCF **task + context**. For multi-day horizons, planning agents decompose, sequence, monitor, adapt — **hierarchical decomposition** is the named strongest strategy — choosing **symbolic** planning (STRIPS, PDDL: guarantees, heavy pre-modeling) vs. **LLM-powered dynamic** (adaptability, no guarantees) by domain stability (Ch 5, p. ~163).
**Trade-off**: CoT is cheap but rigid — an early flawed assumption propagates uncorrected. ToT is high compute with branch-explosion risk. **Static plan execution is not planning** (Ch 5, pp. ~165, ~172).
**Do not attach checkpointing to ToT** — checkpointing belongs to Ch 5's guided-autonomy path (#3).
→ `chapters/ch03-prompting.md`, `chapters/ch05-cognitive-architectures.md`

---

## B. Cognitive architecture

### 3. Autonomous cognitive loop with an escalation gate (Ch 5, pp. ~151–161)
**When**: single-objective tactical tasks needing real-time response — support, triage, alerting.
**How** — exactly five steps: `enhanced_perception → autonomous_reasoning → autonomous_action_execution → learn_from_interaction → update_session_memory`. Learning adjusts autonomy thresholds when `success_score < 0.7`; plans are **dependency-aware task DAGs**, not linear lists. Autonomy level comes from scoring three weighted candidates — full autonomous / immediate escalation / guided (p. ~152); guided means `execute_with_checkpoints()`, and **those checkpoints are human oversight, not crash recovery**. Escalate on any safety violation, `confidence < 0.8`, complexity > 0.7, or business impact > 0.6.
**`autonomous_safety_check()` is NOT a step inside the loop** — it is applied around planned actions (pp. ~159, ~161), returning `(ok, violations)` over financial limits, data-access permissions, impact assessment.
**Trade-off**: responsiveness raises hallucination and unsafe-action risk absent constraints; lacks depth for strategic tasks (p. ~172).
→ `chapters/ch05-cognitive-architectures.md` · `chapter05/resilience.py`

### 4. Three-tier memory (Ch 5, pp. ~167–171; Ch 10, pp. ~321–323)
**When**: any agent needing continuity beyond a single turn.
**How**: **working** (current session, in the context window; loss → incoherence) · **episodic** (cross-session, vector similarity over history; loss → repeated history) · **semantic** (domain facts; loss → stale answers). Flow: search episodic (`limit=3`) → inject → write back. **Consolidation is required** — summarize long episodic entries, or vector retrieval degrades as history grows (p. ~171). **Archival is an explicit operation**, not a side effect of keeping the transcript; semantic memory holds sensitive data (Ch 10, p. ~322).
**Trade-off**: retrieval-design sensitivity and information-overload risk (pp. ~172–173).
→ `chapters/ch05-cognitive-architectures.md`, `chapters/ch10-conversational-content.md`

---

## C. Retrieval and knowledge

### 5. Four-stage RAG pipeline with parallel provenance (Ch 6, pp. ~178–179)
**When**: any agent whose answers must be factually anchored and auditable.
**How** — **four** stages: **query understanding** → **retrieval** → **preprocessing** → **synthesis** (answer *using only the provided sources*). Workflow choice: single-stage for narrow latency-sensitive queries, multi-stage for open-ended aggregation, hybrid keyword+vector when the corpus mixes structured terminology with prose (p. ~179).
**Provenance is a parallel component — not a fifth stage, not a final step.** Citations, metadata, and confidence are collected continuously across all four; bolted on at the end it cannot make the answer auditable. There is **no "Advanced RAG" five-component variant** in this book; do not invent one.
**Trade-off**: provenance plumbing across every stage, for an answer an auditor can accept.
→ `chapters/ch06-retrieval-knowledge.md` · `chapter06/agent_utils.py`, fixtures `chapter06/docs/knowledge_base_rag.txt`

### 6. Chunking, retrieval tuning, failure diagnosis (Ch 6, pp. ~182–183; Ch 2, pp. ~83–84)
**When**: always — chunking is "the most consequential configuration decision in a RAG system."
**How**: **fixed-size** for uniform documents · **recursive** is the recommended default for mixed corpora · **semantic** for narrative text at higher ingestion cost. Reference stack `chunk_size=1000, chunk_overlap=200`, FAISS, `k=3` — explicitly "not arbitrary". **Tuning levers are Ch 2, pp. ~83–84**, not Ch 6: hierarchical chunking; **reranking** (**Cohere Rerank**, **Sentence Transformers cross-encoders**); metadata filtering. Diagnose with `return_source_documents=True`.
**Distinct signature**: **uniformly low similarity scores across all chunks = vocabulary mismatch.** The fix is lexical — **add BM25 hybrid retrieval**. It is **not** a bigger embedding model.
**Trade-off**: misconfigured chunking is the most common source of retrieval-quality degradation in production.
→ `chapters/ch06-retrieval-knowledge.md`, `chapters/ch02-toolkit.md`

### 7. Authority-weighted retrieval with a citation gate (Ch 14, pp. ~439–449)
**When**: retrieval must preserve source hierarchy (binding vs. persuasive, current vs. superseded) and outputs cite external sources.
**How**: dense + sparse retrieval under jurisdiction/authority filters, re-ranked on **0.5 × semantic similarity + 0.3 × authority + 0.2 × recency** — **authority and recency are tie-breakers, not primary signals.** **Citation gate**: verify every citation against an authoritative KB before it enters the artifact; flag unverifiable ones in place; emit `citations_verified` (all-or-nothing) and a `quality_score`.
**Trade-off**: verification latency. Hallucinated case law is "the single most dangerous failure mode", and **the consequences fall on the human, not the algorithm** (Schwartz sanction, p. ~442).
→ `chapters/ch14-financial-legal.md`

---

## D. Tools and orchestration

### 8. Tool-using agent — registry, schemas, selection funnel (Ch 7, pp. ~206–213; Ch 8, pp. ~235–241)
**When**: any agent invoking external functions.
**How**: Think → Plan → Act over four decoupled parts — **reasoning core** · **tool registry** (name, description, I/O schemas, **operational status**) · **execution engine** (state, retries, errors — decoupled from reasoning) · **tool chest** (single-responsibility functions with their own timeouts and validation). **Schemas are contracts, not documentation**; one `data_state` carried tool-to-tool makes multi-step processes coherent (p. ~217). Once the registry outgrows the prompt add the **selection funnel** — intent classification → semantic search (**discard below ~0.7**) → constraint filtering (Figure 7.2). Related: the LLM interprets intent, **Pandas/NumPy/Statsmodels do the calculation** (Ch 8).
**Trade-off**: upfront boilerplate; kills silent failures from malformed arguments and lets tools be added without touching core logic.
→ `chapters/ch07-tool-orchestration.md`, `chapters/ch08-data-reasoning.md` · `chapter08/utils.py`

### 9. Layered failure recovery (Ch 7, pp. ~215–216; Ch 4, pp. ~132–135)
**When**: always. "Failure is not a possibility; it is an inevitability."
**How** — tool level, six layers: safe invocation wrappers · fallback tool chains · confidence-based switching · **failure memory** (circuit breaker marking a repeatedly failing tool unavailable) · escalation with full context · telemetry. Of the four failure modes, **semantic mismatch** (tool succeeds, misses the intent) passes every runtime check. Infra level (Table 4.1): circuit breakers, bulkheads, timeout+retry, failover to cached responses. **Gotcha** (p. ~135): under Kubernetes circuit state must live in a **shared Redis key**, not module-level globals.
**Trade-off**: layered recovery beats prevention; without failure memory the agent burns cycles re-selecting a down tool.
→ `chapters/ch07-tool-orchestration.md`, `chapters/ch04-deployment.md` · `chapter07/helpers/resilience.py`, `chapter05/resilience.py`, `chapter15/resilience.py`, `chapter16/resilience.py`

### 10. Chain-of-agents orchestration (Ch 7, pp. ~217–225; Ch 14, pp. ~424–434)
**When**: tasks spanning multiple domains where deep per-domain specialization pays.
**How**: a **manager/supervisor** coordinates specialists — each granted only the tools its role needs — on four pillars: defined roles, communication infrastructure, **shared context as the single authoritative workspace**, execution orchestration. MCP and A2A standardize the wire level. Nine-field message envelope: `sender_id`, `recipient_id`, `message_type`, `timestamp`, `confidence_score`, `data_payload`, `context_references`, `requires_response`, `priority_level` (Ch 3, pp. ~117–118; Ch 14, p. ~438). Conflict arbitration: detect (similarity below ~0.7) → arbitrate → confidence consensus → human escalation (Figure 7.4).
**Gate at the supervisor, not inside specialists** (Ch 14, p. ~432): scattered checks mean no single point guarantees the gate ran. **State-graph routing is the architectural safeguard** — it makes tool permissions, human checkpoints, and audit trails enforceable.
**Trade-off**: orchestration complexity; each specialist becomes independently optimizable and testable.
→ `chapters/ch07-tool-orchestration.md`, `chapters/ch14-financial-legal.md` · `chapter14/mock_data.py`

### 11. Bounded multi-agent deliberation (Ch 15, pp. ~473–484)
**When**: a defensible synthesis matters more than a single answer.
**How**: **independent proposals → cross-evaluation → synthesis.** Each agent evaluates proposals **that are not its own** — no self-scoring — on correctness, completeness, feasibility, risks (0–10). Expertise-weighted mean; **no relevant evaluations scores 0.0**, signalling missing coverage rather than defaulting silently. Bounded by `max_rounds=3` — **at the limit it still synthesizes.**
**Diversity must be engineered** (pp. ~472, ~483–484): the **Condorcet Jury Theorem** requires independence, and agents sharing a base model have **correlated errors**. Three guards — a rotating **adversarial critic**, **randomized proposal order**, a **relevance score gating** voting weight.
**Trade-off**: coordination overhead. A system that merely *averages* independent answers does not earn it back.
→ `chapters/ch15-education-knowledge.md`

---

## E. Safety, verification, compliance

### 12. Defense-in-depth for agents — five controls (Ch 4, pp. ~137–140)
**When**: any agent processing user-provided content or invoking tools.
**How** — **five controls, not three layers**: (1) **input validation** — isolate user input from system commands; (2) **prompt schema enforcement** — typed I/O constraining reasoning boundaries; (3) **memory governance** — vet persistent-memory writes, scope by session or role; (4) **tool gating** — whitelist tools, enforce parameter constraints at runtime; (5) **interface hardening** — rate limiting, auth, sandboxing. Defends a three-level threat taxonomy (Table 4.3a): input, execution, memory. Zero-trust companion (Table 4.4): least privilege **per task**, verification **per invocation, not per session**, micro-segmentation, immutable containers. Audit retention **90–180 days** is an operational floor; regulated domains impose far longer (#17).
**Trade-off**: processing steps per request. "Security and privacy in AI agents cannot be retrofitted" (p. ~140).
→ `chapters/ch04-deployment.md` · `chapter04/agent_utils.py`

### 13. Safety and ethics upstream of generation (Ch 12, pp. ~360–380; Ch 10, pp. ~319–323)
**When**: consequential decisions, or any high-stakes conversational domain.
**How**: `Perceive → Reason → Plan → [ETHICAL CHECKPOINT] → Execute`, with a value alignment framework, fairness evaluator, bias detector, explanation generator. **Value hierarchy is non-negotiable: safety constraints (hard stops) > ethical principles > task objectives.** Conversational analogue (Ch 10): the safety layer is a **deterministic circuit breaker at the entry point** — a crisis trigger diverts to a fixed protocol, **bypassing the cognition core**. Fairness: parity, equal opportunity, and predictive parity cannot all hold unless base rates are identical, so choose which governs — **the choice is normative, not technical** (Table 12.1, pp. ~368–369). Explainability: **SHAP** global, **LIME** per-prediction, **different explanations per audience** without distorting the finding (p. ~380; Ch 13, pp. ~404–405).
**Trade-off**: latency per decision. **Ethics is structural, not behavioral** — an output filter is not an ethical agent, and a prompt instruction is not a safety layer.
→ `chapters/ch12-ethical-explainable.md`, `chapters/ch10-conversational-content.md` · `chapter12/ethical_core.py`, `chapter12/explainability_core.py`

### 14. Verification & Validation meta-reasoning layer (Ch 8, pp. ~242–246; Ch 11, p. ~343)
**When**: hallucinated statistics or unsupported claims carry legal or reputational risk.
**How**: the primary agent solves; the validation layer examines **how** the solution was produced. **Fact-checking** — decompose into discrete claims, test each against retrieved evidence, classify with **NLI models** as Supports / Refutes / Neutral. **Logical coherence** — decompose the chain into a claim graph, flag nodes with empty or self-referential support. Plus **adversarial red-teaming with a second LLM**. Perceptual analogue: cross-check VLM output against **deterministic CV models** (Ch 11, p. ~343). **Route failures, don't discard them** — failed verifications go to open-ended reasoning, and no unverified claim reaches a decision-maker unflagged (pp. ~262–263).
**Trade-off**: extra LLM calls and retrieval per claim.
→ `chapters/ch08-data-reasoning.md`

### 15. Calibrated confidence and confidence routing (Ch 13, pp. ~401–404; Ch 6, pp. ~184–191)
**When**: confidence scores drive downstream action.
**How**: a calibrated agent's confidence reflects actual accuracy — at 80% stated, roughly 80% correct. Guarantee via **Brier score decomposition** (Reliability / Resolution / Uncertainty); **Platt or temperature scaling minimizes the Reliability term specifically**. Downstream it is a pipeline control signal: in document extraction it triages, gates fallbacks, and routes to human review — **discarding OCR confidence after the OCR stage breaks the whole downstream pipeline** (Ch 6).
**Derive the threshold, don't guess it**: the book's `escalation_threshold=0.15` came from **cost-asymmetry analysis** — a missed case costs roughly an order of magnitude more than a false alarm, putting the boundary in the **0.12 to 0.18 band; 0.15 is the midpoint, rounded toward the conservative end**. Below threshold on non-critical conditions the system **presents no diagnosis at all** — it flags for expert review.
**Trade-off**: uncalibrated confidence is a misleading signal, worse than none.
→ `chapters/ch13-healthcare.md`, `chapters/ch06-retrieval-knowledge.md` · `chapter06/samples/sample_invoice.png`

### 16. Gates in the control flow — validation and human approval (Ch 14, pp. ~436–447; Ch 9, pp. ~286–308; Ch 10, pp. ~325–335; Ch 7, pp. ~228–231)
**When**: no non-compliant, off-brand, or unapproved output may reach the user.
**How**: a gate node inside the state machine that every output must pass — `produce → validate`, conditional `validate → {deliver, revise}`, `revise → validate`, `deliver → END` — running until constraints pass, **before any output reaches a downstream channel**. Four flavours, same shape: **compliance** (Ch 14) checks suitability and concentration at the data-access layer, running **in parallel at every stage as a gate, not as post-processing**, with the regulation set a **caller-supplied context parameter** · **policy-as-code** (Ch 9) uses **OPA/Rego** as "the test suite for compliance" plus PII/PHI data-flow analysis, hard failure blocking the merge · **editorial** (Ch 10) makes the Editor a **critic, not a creator** · **human approval** (Ch 7) uses explicit guards (**`confidence_score < 0.85`** → Pending Human Review) with waiting states and rollback paths, risk-tiered so low auto-deploys and high needs cross-functional sign-off (Ch 9). **Human review is a planned branch in the control flow, not an exception handler**; the target is **conditional autonomy — the agent proposes, the human disposes.**
**Trade-off**: revision cycles and throughput. "It is structurally impossible for a non-compliant recommendation to reach the client."
**Scope caution for Ch 14**: it names **no statute beyond the suitability obligation** — MiFID II and FCA COBS belong to the **Ch 3 case study (pp. ~119–120)**, not here; nor do FINRA, SEC, or Reg BI. Ch 14 does **not** prohibit autonomous execution — it explicitly describes "in fully autonomous setups, execute trades directly" (p. ~434). Its force-multiplier position (p. ~449) is an effectiveness claim, not a legal prohibition.
→ `chapters/ch14-financial-legal.md`, `chapters/ch09-software-dev.md`, `chapters/ch10-conversational-content.md`, `chapters/ch07-tool-orchestration.md` · `chapter09/compliance_engine.py`

### 17. Regulated-domain architecture — isolation, provenance, audit (Ch 13, pp. ~392–405)
**When**: any domain where a decision will be audited.
**How**: four layers — **data ingestion** → **domain knowledge** (versioned, provenance-tracked) → **reasoning and decision** → **explanation and delivery**. Each exposes **only a typed interface**, so a guideline database updates without touching diagnostic logic and one trace produces a clinician report, a patient summary, or a regulator record. **Feedback loops must be narrow and explicit, not general backpropagation channels**. Priority: verifiability, explainability, graceful degradation, then speed (p. ~392). Per-result provenance carries `source`, `version`, `retrieved_at`, `confidence`; **currency is a safety property**, and on contradicting sources the agent **does not silently choose** but flags both and defers (pp. ~395–397). Privacy by architecture: edge inference, de-identified features only, **differential privacy at epsilon = 1.0** (pp. ~405–406).
**Immutable audit** (pp. ~403–405): inputs (identifiers encrypted), KB version, model version, reasoning trace, recommendation, confidence breakdown, safety alerts — **cryptographically signed, append-only, retained a minimum of 7 years to satisfy HIPAA.**
**Trade-off**: "the model thought so" is not an answer to an auditor. The separation is a regulatory necessity, not engineering convenience.
→ `chapters/ch13-healthcare.md` · code: **none — Ch 13 ships no `.py` modules**; MockLLM is inline in its notebook, and `DrugInteractionDB` / `ClinicalGuidelineEngine` / `DiseaseOntology` / `MedicalLiteratureIndex` are illustrative interface stubs.

---

## F. Deployment, cost, resilience

### 18. Infrastructure matched to agent typology (Ch 4, pp. ~126–136)
**When**: before any deployment decision — retrofitting infrastructure onto a misaligned architecture is explicitly expensive.

| Agent type | Execution | Target | Coordination |
|---|---|---|---|
| Reactive | Stateless functions | Serverless/edge | Event trigger (HTTP/SQS/webhook) |
| Deliberative | Stateful, compute-bound | GPU VMs / containers | Planning DAGs, checkpointing |
| Hybrid | Context-aware multistage | Microservice clusters | Internal message bus, fallback |
| Multi-agent | Distributed, autonomous | Kubernetes/mesh + Kafka | Messaging, vector context, roles |

Decompose into Planner · Retriever · Memory Store · Execution Engine · Response Synthesizer, **orchestration stateless and reasoning state external** (Table 4.2). **Rollback spans four substrates** (p. ~136): model weights, tool configurations, memory contents, conversation history — a rolling update reverts only the container, and **tool API versions must be explicitly pinned.** Instrument A/B tests beyond latency: **LLM behavioral regressions often produce no latency or error anomaly.**
**Trade-off**: agents **scale on cognitive load** — complexity, memory state, reasoning depth, tool dependency — not request volume.
→ `chapters/ch04-deployment.md` · `chapter04/agent_utils.py`

### 19. Cost and latency tiering (Ch 4, pp. ~129–132; Ch 13, p. ~404; Ch 15, p. ~469)
**When**: any production agent. The five strategies interlock — neglecting one undermines the whole.
**How**: model routing with **confidence-based escalation upward** · tiered architecture (rules → mid model → expensive model) · caching · budget enforcement with **graceful degradation** · cost monitoring. Tag every call with `task_id`, `agent_type`, `cost_tier`. Cost vectors include **inter-agent messaging with potentially quadratic growth**. Concrete cascade: a rule classifier catches ~70% in under 50 ms and the LLM fires only on no-match or `confidence < 0.7`, taking latency **8 s → 2.5 s** (Ch 15, p. ~469; explicitly generalizes beyond education). Latency budget: reactive **sub-second**, deliberative **3–5 s**, over budget return a **partial result plus an explicit incompleteness flag** (Ch 13, p. ~404).
**Trade-off**: routing infrastructure. Guiding principle is **value per unit of compute**.
→ `chapters/ch04-deployment.md`, `chapters/ch13-healthcare.md`, `chapters/ch15-education-knowledge.md`

---

## G. Generation and improvement loops

### 20. Test-Driven Generation with bounded refinement (Ch 9, pp. ~270–308)
**When**: generating code that must be verifiably correct, not plausibly correct.
**How**: **Red** — a tester agent generates a failing suite from the requirement; **the suite is the executable specification**. **Green** — a developer agent writes minimal passing code, iterating on structured test failure output. **Refactor** — clean up, re-run. The refinement prompt is state-rich (spec + latest code + stack trace + prior failed attempts), and **the LangGraph conditional edges hardcode `iterations < 3`** (p. ~284). Self-improvement extends the same loop: sensing → critic scoring KPIs → planner emitting a typed `ImprovementHypothesis` → **risk-tiered HITL** (#16) → learning layer, guarded by immutable version history and **drift detection via embedding-distribution shift** (pp. ~296–308).
**Trade-off**: multiple LLM calls per task; without a cap, refinement spins forever. Full fine-tuning risks **catastrophic forgetting**; unversioned learning makes rollback and audit impossible.
→ `chapters/ch09-software-dev.md` · `chapter09/agent_nodes.py`, `chapter09/state_models.py`, `chapter09/self_improving.py`

---

## H. Physical world and partial observability

### 21. Asymmetric control — four-layer hierarchy, admissible envelope (Ch 16, pp. ~489–521)
**When**: any agent whose actions move physical matter, or any go/no-go spanning several independent domains.
**How**: split an **asynchronous reasoning layer** (LLM, plans from intent) from a **synchronous deterministic controller** (real-time execution enforcing hard constraints). The handoff is a bounded plan — explicit **1–10 second look-ahead**, each action inside an **action envelope** (position, velocity limit, force ceiling, workspace boundary). Out-of-envelope commands are **rejected before execution** and trigger replanning.
**The stack is four layers, not two** (Table 16.1):

| Layer | Output | Frequency | Mechanism |
|---|---|---|---|
| Task planning | Symbolic goals | **0.1–1 Hz** | PDDL planners / LLM reasoning |
| Motion planning | Collision-free paths | **1–10 Hz** | RRT, PRM, optimization |
| Trajectory control | Time-parameterized path | **50–200 Hz** | PID / MPC |
| Servo control | Motor currents | **1–10 kHz** | Current and torque loops |

**The LLM interacts only at the task-planning level; the lower three have no LLM involvement.** In LangChain this is **tool-mediated indirection** (`query_world_model`, `dispatch_motion_command`, `check_safety_constraints`) so the reasoning layer cannot bypass the safety layer (p. ~499). The world model holds **inferred object states, never raw camera frames or point clouds** (pp. ~497–498).
**A_safe(s)** (pp. ~492, ~497): the safety layer sits **between the control interface and the actuators** — an **explicit restriction on the executable action set**, not an emergent property of correct planning. **Hardware e-stop** is the unconditional override, outside the command pipeline; human in the perimeter → halt **within 100 ms**. Scaled to multi-domain go/no-go, **a single domain returning unsatisfied vetoes the whole envelope**, and **staleness is failure** — STALE counts as a failed constraint (pp. ~512, ~519–521).
**Trade-off**: architectural complexity. **The high layer selects WHAT under uncertainty; the low layer determines HOW deterministically. Never mix.**
→ `chapters/ch16-embodied.md` · `chapter16/mock_layer.py`, `chapter16/resilience.py`

### 22. Belief state over hidden variables — POMDP decomposition (Ch 15, pp. ~453–454; Ch 16, p. ~493)
**When**: the critical variable is hidden and observable only through noisy proxies — a learner's mastery, a partially observable workspace.
**How**: exact POMDP solutions are intractable at scale, so decompose into belief estimation + action selection + temporal planning. **Ch 15 (education)**: Bayesian Knowledge Tracing maintains `b(s)` per skill with slip and guess as the observation model — which is why **"correct" ≠ "mastered"** — and a curriculum planner selects by expected learning gain using ZPD heuristics. **Ch 16 (physical)**: **particle filters**, or **extended Kalman filters** for continuous near-linear dynamics, both at **10–100 Hz** without blocking planning.
**Act across the belief, not on the mode**: pick actions performing well **across plausible states, not just the most likely one** — a warehouse robot slows near blind corners because the belief assigns non-trivial probability to a human out of sight.
**Trade-off**: you lose the optimality of an explicit value-iteration solver and gain individually testable, efficient modules.
→ `chapters/ch15-education-knowledge.md`, `chapters/ch16-embodied.md`

---

## I. Multimodal perception

### 23. Vision-Language triad (Ch 11, pp. ~339–343)
**When**: reasoning must be grounded in pixel-level evidence rather than a textual description of an image.
**How**: **visual encoding** (a ViT emits visual tokens from fixed-size patches) → **alignment** (linear projections, MLPs, or **Q-Former** — it "determines whether the language model can genuinely 'see' the image or merely receives a degraded signal") → **LLM** grounding reasoning through cross-modal attention. Encoder choice: **SigLIP** for language-aligned retrieval, **DINOv2** for dense prediction — but **foundation model quality, not just the encoder, is the primary driver** (p. ~341). Integration is adapter-based, cross-attention, or early fusion; **visual token count is the lever behind both latency and early-fusion overhead** (pp. ~342–343).
**Trade-off**: reasoning over textual descriptions incurs inevitable information loss. Pair with the deterministic validation layer (#14).
→ `chapters/ch11-multimodal.md` · `chapter11/mock_backends.py`, `chapter11/agent_logger.py`

---

## Cross-cutting invariants

1. **Safety and compliance are structural, not behavioral.** An output filter is not an ethical agent (Ch 12, p. ~360); a prompt instruction is not a safety layer (Ch 10, p. ~320); safety is an explicit restriction on the admissible action set (Ch 16, p. ~497).
2. **Provenance and audit trails are load-bearing.** Provenance runs *parallel* to retrieval, never as a final step (Ch 6, p. ~179); every result carries source, version, timestamp, reliability (Ch 13, p. ~396).
3. **Escalation is a design commitment.** The cost of a wrong autonomous action exceeds the cost of routing to a human, and that judgment lives in explicit threshold logic — not in the model (Ch 5, p. ~161; Ch 13, p. ~419).
4. **Bound every loop.** Refinement caps at 3 (Ch 9, p. ~284); consensus caps at `max_rounds=3` and still synthesizes at the limit (Ch 15, p. ~481); failure memory stops re-selection of a down tool (Ch 7, p. ~215).
5. **Confidence is only useful when calibrated**, and thresholds should be *derived* from cost asymmetry rather than guessed (Ch 13, p. ~404).
6. **Log every tool call, decision, confidence score, and rationale** — simultaneously the debugging surface and the compliance audit trail (Ch 7, pp. ~216, ~231).
