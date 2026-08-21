# Cheatsheet — 30 Agents Every AI Engineer Must Build

Decision tables only. **Every row is book-sourced.** Citations use the EPUB page anchor
(`id="page_NNN"`) = printed page +31. If the book doesn't say it, it isn't here.
Everything cut from this file is indexed in §14.

---

## 1. Framework selection (Table 2.1, Ch 2, p. ~69)

| Framework | Strengths | Limitations | Ideal use case |
|---|---|---|---|
| LangChain | Modular design, broad integrations | **No native multi-agent support**; abstractions add latency overhead (p. ~77) | LLM pipelines, tool workflows |
| LlamaIndex | Advanced retrieval, semantic compression | Requires orchestration support | Document Q&A, memory layers |
| **AutoGPT** | Autonomous goal planning | Low reliability, fragile control | Research prototypes |
| CrewAI | Role-based coordination | **Early-stage maturity** | Experimental multi-agent, sandboxed collaborative prototypes |

Chapter text, not Table 2.1: **LangGraph** — DAG, parallel branches, state + observability; named for **deterministic control in regulated industries** (pp. ~70, ~75, ~78). **AutoGen** — LLMs as conversation participants, explicit turn-taking and stop conditions (p. ~77).
Compose, don't select (p. ~78): LangChain + LlamaIndex · CrewAI + LangChain · LangGraph where regulated.

---

## 2. LLM selection (Ch 2, p. ~79)

**Criteria, not a model shortlist:**

| Axis | What the book says |
|---|---|
| Capability vs. cost | Lightweight/fast, weaker on complex reasoning (**Mistral 7B**) ⟷ highly capable, compute-intensive (**GPT-4, Claude 3, Gemini**) |
| Task specialization | Some models excel at coding, others at creative content or multi-turn reasoning |
| Context window | Ranges **8K to over 1M tokens** |
| Ops | Hosting, pricing models, rate limits vary significantly |
| Licensing | **Open weight** (local deploy + fine-tune) vs. **closed source** (API, managed) |

**Benchmark on YOUR task**, never generic leaderboards. **Token normalization**: identical text yields different token counts per tokenizer — hits cost *and* context limits.
Hybrid `route_to_model`: FACTUAL → Mistral 7B · CREATIVE → Claude · ANALYTICAL → GPT-4o.

> **There is no "GPT-4o / Claude 3.5+ Sonnet / Haiku / 200K context" selection table in this book.**
> *Sonnet*, *Haiku*, and *200K* have **zero occurrences**. `GPT-4o` and `Claude 3.5+` appear once, in
> the **Preface** *Software/hardware covered* list (p. ~25) — a code-dependency list, not guidance.

---

## 3. Memory selection (Ch 5, p. ~167)

| Type | Where it lives | Trigger condition | Failure if skipped |
|---|---|---|---|
| Working | **Directly in the LLM prompt**; cleared at session end / token limit | Active session, current turn | Loses coherence; forgets earlier turns |
| Episodic | Timestamped interaction history; vector similarity search | Cross-session continuity; user cites prior interaction | Repeats history; personalization breaks |
| Semantic | Structured facts from docs/APIs/DBs; keyword or semantic search | Domain knowledge not answerable from session | Hallucinated or stale, ungrounded claims |

Vector stores (pp. ~168–169): **Chroma** local dev · **Pinecone** managed production scale · **Weaviate** open source hybrid keyword+vector. Reference impl retrieves `limit=3` episodic memories.
**Consolidation is mandatory** — summarize long episodic entries or retrieval degrades as history grows (p. ~171).

---

## 4. Cognitive loop

Base loop (Ch 1, p. ~36; code Ch 5, p. ~155):

```
Perceive → Reason → Plan → Act → Learn
   ↑                              |
   └────────── Feedback ──────────┘
```

**The safety check is NOT inside the base loop.** The ethical variant *inserts* a gate (Ch 12, p. ~360):

```
Perceive → Reason → Plan → [ETHICAL CHECKPOINT] → Execute
```

Ch 5's analogous `autonomous_safety_check()` wraps planned actions as a *design consideration* — not a step inside `cognitive_loop()` (Ch 5, pp. ~159, ~161).

---

## 5. Agentic AI Progression Framework (Ch 1, p. ~63)

| Level | Name | Defining property |
|---|---|---|
| 0 | Manual operations | No automation or intelligence; software is a passive tool |
| 1 | Reactive agents | Deterministic trigger → preprogrammed action; stateless, no learning |
| 2 | Tool-using agents | Parses NL, selects tools by context, chains ops; bound to session context |
| 3 | Planning agents | Decomposes goals, uses intermediate feedback, re-plans, persistent awareness |
| 4 | Learning agents | Evolves from experience; per-user models; continuous strategy refinement |

Scored on **autonomy, reasoning, adaptability**. Don't skip levels (p. ~63).
Do not conflate with the **Five Levels of Agent Interaction**: Direct LLM → Proxy agent → Assistant system → Autonomous agent → Multi-agent system (Ch 1, p. ~53).

---

## 6. Chunking strategy (Ch 6, p. ~182)

| Strategy | Use for | Note |
|---|---|---|
| Fixed-size | **Uniform, well-formatted documents** | Simplest; fixed character/token boundary |
| Recursive | Mixed-content corpora — **recommended default** | Paragraphs → sentences → words; char-level only as fallback |
| Semantic | Narrative text needing highest fidelity | Embedding similarity detects topic shifts; higher ingestion cost |

| Size band | Effect |
|---|---|
| 200–500 chars | Better retrieval precision; risks omitting surrounding context |
| 1,000–2,000 chars | Richer context; dilutes embedding signal, reduces recall |
| Overlap 200 on chunk 1,000 | A sentence spanning two chunks is fully captured by at least one |

**"The most consequential configuration decision in a RAG system"**; misconfiguration is the top source of production retrieval degradation.
Reference stack (pp. ~180–181): `RecursiveCharacterTextSplitter(1000, 200)` · `text-embedding-3-large` · `gpt-4o-mini` `temperature=0` · `RetrievalQA(return_source_documents=True)` · `k=3`. Also named: **hierarchical chunking** (Ch 2, p. ~83).

---

## 7. Retrieval failure → root cause (Ch 6, p. ~183)

| Symptom | Root cause | Fix |
|---|---|---|
| Vague/off-topic answer; top chunks from the wrong document | Required document missing from the index | Re-ingest it |
| Chunks relevant but lack the information to answer | Chunk boundaries split the clause | Resize so the clause stays a **discrete unit** |
| Wrong document class surfaced | No scoping filter | `filter={"doc_type": "subscription_policy"}` |
| **Similarity scores uniformly low across all chunks** | **Vocabulary mismatch** — query terminology absent from the embedded corpus | **Add keyword (BM25) hybrid retrieval** — the fix is lexical, not generative |
| Unverifiable source | Provenance metadata missing or mismatched | Restore provenance capture across the pipeline |

> **Not "embedding model mismatch → align models".** Ch 6, p. ~183 attributes uniformly-low scores to
> **vocabulary mismatch**, fixed by BM25 hybrid retrieval, and warns explicitly against reaching for a
> bigger embedding model on this signature.

---

## 8. Tool failure → recovery (Ch 7, pp. ~215–216)

| Failure mode | Recovery |
|---|---|
| Input validation error | Safe invocation wrapper; schema rejects before execution |
| Runtime failure | Targeted retry (exponential backoff for network errors) |
| **Semantic mismatch** — tool succeeds, result misses intent | Confidence-based switching or human review; runtime checks will not catch it |
| Tool unavailability | Fallback chain (`..._api_A` → `..._api_B`); **failure memory** marks a repeatedly failing tool unavailable |
| Anything unresolved | Escalation handing a human full context; logging and telemetry throughout |

**The registry schema is enforcement, not documentation** (pp. ~206, ~208).

---

## 9. Defense in depth — **five** layered controls (Ch 4, p. ~140)

1. **Input validation** — strip malicious tokens, enforce structured prompts, isolate user input from system commands
2. **Prompt schema enforcement** — typed I/O definitions constraining reasoning boundaries
3. **Memory governance** — vet updates to persistent memory, constrain scope by session or role
4. **Tool gating** — whitelist allowed tools, enforce parameter constraints at runtime
5. **Interface hardening** — rate limiting, authentication, sandboxing on exposed endpoints

> Five controls. Not three, and not a named "stack" — the book presents them as layered controls
> across cognitive and operational boundaries.

---

## 10. Thresholds the book actually specifies

Use this instead of inventing numbers. Everything here is a book-stated value.

| Threshold | Value | Context | Cite |
|---|---|---|---|
| Escalation — confidence | `confidence < 0.8` | One of five weighted escalation factors | Ch 5, p. ~160 |
| Escalation — complexity | `assess_issue_complexity > 0.7` | Same weighted score | Ch 5, p. ~160 |
| Escalation — business impact | `calculate_business_impact > 0.6` | Same weighted score | Ch 5, p. ~160 |
| Escalation — safety | **Any** safety violation | Same weighted score | Ch 5, p. ~160 |
| Escalation — user preference | `check_escalation_preference(user_id)` | Same weighted score | Ch 5, p. ~160 |
| Learning trigger | `success_score < 0.7` → adjust autonomy thresholds, flag for model improvement | Post-interaction learning | Ch 5, p. ~155 |
| Support automation ceiling | "up to **80%** of user queries" handled autonomously | Claim about support query volume, **not** a general design target | Ch 5, p. ~162 |
| Tool semantic-search cutoff | Confidence threshold such as **0.7** | Tool selection funnel stage 2 | Ch 7, p. ~211 |
| Conflict detection | Semantic similarity below a threshold such as **0.7** | Multi-agent arbitration | Ch 7, p. ~224 |
| Arbiter consensus | Policy threshold such as **95%** | Confidence-based consensus | Ch 7, p. ~224 |
| Conflict score (worked code) | `abs(sentiment − stock_change/10) > 0.5` (≈5-point gap) | Market intelligence example | Ch 7, p. ~225 |
| Claims HITL gate | `confidence_score < 0.85` → Pending Human Review | Insurance claims state machine | Ch 7, p. ~231 |
| Circuit breaker | `FAILURE_THRESHOLD = 3` | Tenacity + manual breaker example | Ch 4, p. ~135 |
| Audit retention (general) | **90–180 days**, encrypted append-only | Security incident preparedness | Ch 4, p. ~139 |
| Circuit-breaker payoff | P99 latency cut **40–60%** during failures | Resilience patterns | Ch 4, p. ~132 |
| OCR gating | `CONFIDENCE_THRESHOLD = 60` on Tesseract's 0–100 scale | Document intelligence | Ch 6, p. ~185 |
| Doc-intelligence accuracy | **95%** field-level on labeled validation set | ADL target (invoice no., totals, dates, vendor) | Ch 6, pp. ~190–191 |
| Doc-intelligence HITL | Human review kept **under 8%** | ADL target | Ch 6, p. ~191 |
| Clinical escalation | `escalation_threshold = 0.15` (derived range **0.12–0.18**, midpoint rounded conservative) over MI, PE, sepsis, stroke | Cost-asymmetry analysis, ~1 order of magnitude | Ch 13, pp. ~401, ~404 |
| Clinical latency budget | Reactive tier **sub-second**; deliberative tier **3–5 s** target; degrade to **two highest-confidence diagnoses** | Tiered processing | Ch 13, p. ~404 |
| Clinical audit retention | **Minimum 7 years**, cryptographically signed, append-only — HIPAA | Audit and retention | Ch 13, p. ~405 |
| Financial risk bands | Composite ≥ **7.0** HIGH, ≥ **4.0** MODERATE, else LOW (weights 0.4 vol + 0.35 drawdown + 0.25 VaR, 0–10 scale, 90-day lookback) | `RiskScorer` | Ch 14, pp. ~431–432 |
| Financial tier classifier | `abs(price_change) > 5` High, `> 2` Moderate, else Low | Baseline on Finnhub `dp` | Ch 14, p. ~430 |
| Concentration limit | `max_concentration` default **0.25** | Compliance gate | Ch 14, p. ~436 |
| Education mastery | **0.8** in `CurriculumPlanner`, **0.85** in `StudentModel.get_mastered_objectives()` | "Ready" definition | Ch 15, p. ~457 |
| ZPD offset | δ typically **0.1–0.3** above current mastery (code: `delta = 0.2`, `sigma = 0.25`) | Expected learning gain Gaussian | Ch 15, pp. ~455, ~458 |
| Learning objective size | Roughly **15–45 minutes** of instructional time | Knowledge-graph granularity | Ch 15, p. ~456 |
| Robot human-safety halt | Halt within **100 ms** of detecting a human inside the perimeter | Hard latency guarantee | Ch 16, p. ~492 |
| Belief-state filter rate | **10–100 Hz** (particle / extended Kalman), non-blocking | POMDP filtering | Ch 16, p. ~493 |
| Plan look-ahead horizon | **1–10 seconds** at the task-planning rate | Bounded plan parameters | Ch 16, p. ~490 |
| Drone energy envelope | SoC ≥ **30%** at departure, never projected below **20%** at any waypoint, **15%** reserve at RTH | Unified Constraint Envelope | Ch 16, p. ~512 |
| Influence propagation cutoff | `threshold = 0.1` with multiplicative attenuation | Cross-domain graph traversal | Ch 16, p. ~506 |
| Refinement iteration cap | `iterations < 3` | TDG LangGraph conditional edge | Ch 9, p. ~284 |
| Anomaly cutoff (stats) | `\|z\| > 3` | Data analysis anomaly detection | Ch 8, p. ~241 |

---

## 11. POMDP — hidden state per domain (do not mix them)

| Domain | Hidden state | Observations from | Cite |
|---|---|---|---|
| Healthcare | **True disease state** | Findings weighted by per-condition sensitivity/specificity; priors = epidemiological prevalence + demographics | Ch 13, pp. ~393, ~400–401 |
| Education | **Student's true mastery** | Quiz responses, code submissions, time-on-task, help-seeking via BKT slip/guess; priors = IRT 2PL placement test | Ch 15, pp. ~453–454 |
| Embodied | Physical world configuration | Sensor observations; particle filter / EKF at 10–100 Hz | Ch 16, p. ~493 |

Belief update `P(s|o) ∝ P(o|s) · P(s)` (Ch 13, p. ~393). **Exact POMDP solving is intractable — decompose**: BKT = belief estimation · Curriculum Planner = action selection via ZPD heuristics instead of value iteration · Spaced Repetition = temporal planning (Ch 15, p. ~454).
**Plan across the belief distribution, not the single most likely state** — acting on the mode causes dangerous behavior near belief-state decision boundaries (Ch 16, p. ~493).

---

## 12. Control hierarchy for physical agents (Table 16.1, Ch 16, p. ~494)

| Layer | Produces | Frequency | Techniques |
|---|---|---|---|
| Task planning | Symbolic goals | 0.1–1 Hz | PDDL planners / LLM reasoning |
| Motion planning | Collision-free paths | 1–10 Hz | RRT, PRM, optimization |
| Trajectory control | Time-parameterized path | 50–200 Hz | PID / MPC (100 Hz → corrected every 10 ms) |
| Servo control | Motor currents | 1–10 kHz | Current and torque loops |

**The agent interacts only at the task-planning layer; the lower three have no LLM involvement** (p. ~496). High layer selects WHAT under uncertainty, low layer HOW deterministically — never mix (p. ~489).
Safety enforcement sits **between the control interface and the actuators**. Planning operates within `A_safe(s)`; rejected commands return reason + active constraints to the LLM to replan. A **hardware emergency stop** bypasses all software layers (p. ~497).

---

## 13. Regulated-domain gates

| Gate | Rule | Cite |
|---|---|---|
| Compliance placement | Validator at the **data access layer** and as a **graph gate**, never post-processing — "structurally impossible for a non-compliant recommendation to reach the client" | Ch 14, pp. ~436–437 |
| Risk-limit enforcement | At the **supervisor level**, not inside individual agents | Ch 14, p. ~432 |
| Contract findings | Only **HIGH or CRITICAL** surfaced for attorney review | Ch 14, p. ~447 |
| Citation verification | Cross-reference every citation against an authoritative KB; flag unverifiable ones in place | Ch 14, pp. ~443, ~450 |
| Guideline conflict | Agent **does not silently choose**: flag it, present both with evidence bases, defer to the clinician | Ch 13, p. ~397 |
| Below-threshold non-critical | Present **no diagnosis at all** — flag for physician review with the ambiguous findings | Ch 13, p. ~404 |
| Clinician override | Recorded, feeding accountability and model improvement | Ch 13, p. ~405 |
| Feedback loops | Narrow and explicit, **not** general backpropagation — adaptability traded for auditability | Ch 13, p. ~395 |
| Provenance | Every result carries `source`, `version`, `retrieved_at`, `confidence` | Ch 13, p. ~396 |
| Access control | Enforced at the **retrieval level** so regulated docs never enter the context window | Ch 6, p. ~183 |
| HITL framing | Human review is a **planned, first-class branch** in the control flow, not an exception handler | Ch 7, p. ~224 |

**Regulations**: cite only the ones the book names — GDPR, SOC 2 Type II, MiFID II/FCA COBS, CCPA, EU AI Act, NIST AI RMF 1.0, EO 14110, PCI DSS, HIPAA, HL7 FHIR/US Core 6.0, Transport Canada CARs Part IX. Full list with per-mention pages → `chapters/ch04-deployment.md`.
Securities **suitability** is the only affirmative financial regulatory obligation the chapter states (Ch 14, p. ~434). Contract analysis takes its regulation set as a **caller-supplied context parameter** (`context.get("regulations", [])`), not a fixed list (Ch 14, p. ~447).

---

## 14. Pointer index — cut from this file, still in the skill

- **6 agent traits; cognitive-loop derivation** → `chapters/ch01-foundations.md`
- **PTCF template; pattern→PTCF map (3.1); few-shot vs. RAG (3.2); CoT vs. ToT (3.3); prompt→RAG→LoRA ladder; prompt versioning** → `chapters/ch03-prompting.md`
- **Typology→infrastructure (Fig 4.1; scaling axis = cognitive load, not volume); resilience patterns (4.1); microservices (4.2); rollback substrates; threat taxonomy (4.3a/b); zero trust (4.4); cost-tier routing; incident runbooks** → `chapters/ch04-deployment.md`
- **Decision-maker vs. planner vs. memory-augmented (5.1/5.2); STRIPS-PDDL vs. LLM planning** → `chapters/ch05-cognitive-architectures.md`
- **RAG workflow patterns; RAG production checklist; knowledge agent spectrum (6.1); doc-intelligence pipeline and provenance** → `chapters/ch06-retrieval-knowledge.md`
- **Four tool components; selection funnel; Chain-of-Agents protocol (7.1); conflict arbitration ladder** → `chapters/ch07-tool-orchestration.md`
- **Value hierarchy (safety > ethics > task); fairness impossibility theorem (12.1); algorithmic vs. deployment-context fairness; SHAP vs. LIME** → `chapters/ch12-ethical-explainable.md`
- **Full anti-pattern catalogue** → `patterns.md`

---

## 15. Notes on what is *not* in this book

Recorded so nobody re-adds them from memory. Each was checked against the full EPUB text:

- **No temperature guidance table.** The sampling parameter appears only as a code argument — the
  book's own samples use `temperature=0` / `0.0` (RAG, extraction, financial, embodied) and
  `temperature=0.7` (ToT branches Ch 3 p. ~115, marketing content Ch 10). No factual/creative/
  production ranges are ever stated. Do not confuse this with **temperature scaling**, a
  post-training *calibration* method the book does endorse (Ch 13, p. ~401).
- **No latency budgets for robotics** beyond Table 16.1's frequency bands, the 100 ms human-safety
  halt (p. ~492), and the 1–10 s plan horizon (p. ~490).
- **No retrieval, multimodal, or per-modality latency figures** of the "adds X–Y ms" form.
- **No chunk-overlap percentage rule.** Only the absolute pair 1,000 / 200 and the character bands.
- **Zero occurrences**: Sonnet, Haiku, 200K context, KYC, AML, FINRA, SEC, Reg BI, Dodd-Frank.
- **Docling, Azure Form Recognizer, Kinesis** — not present. Kafka *is* (Ch 4, pp. ~127, ~136).
- The **Ch 8 visualization recommender** is keyword-based *as a teaching example only*; the book
  says production should use an LLM classifier — do not cite the keyword rules as guidance.
- **Ch 11 defends raw pixel input**: Vision-Language agents ingest raw visual data rather than
  textual descriptions, because description-based pipelines lose information (Ch 11, p. ~340).
