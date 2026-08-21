# Concept Map — 30 Agents Every AI Engineer Must Build

Routing layer across the 17 chapter files. Built from a global pass after all chapters were verified.

**Citation convention**: `(Ch N, p. ~NNN)` uses the EPUB anchor number (runs +31 vs. the printed page). Do not convert.

**Two link types, never conflated:**
- **Declared** — the book states the cross-reference in text. Sub-type **[numbered]** ("as we saw in Chapter 5") vs. **[positional]** ("in the next chapter…", no number).
- **`(inferred)`** — the conceptual connection is real, the book does not state it.

**Hard finding from the global pass**: Ch1, Ch16 and Ch17 contain **zero numbered cross-chapter references**. Every "Connects To" entry currently in those three files is inferred. Ch6 has exactly one (to Ch8), and that one is dangling — see §4.

---

## 1. Concept index

`INTRO` = where the concept is introduced. `APPLY` = where it is used/extended. Distinguishing the two is what prevents the misattributions this file exists to fix.

### Cognitive loop (perceive → reason → plan → act → learn)

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **INTRO** — the loop itself; feedback from Act returns to Perceive | p. ~36 |
| Ch5 | **INTRO (code)** — `cognitive_loop()`; Figure 5.1 continuous loop; environmental inputs → perception → cognition → action → new state | p. ~154–156 |
| Ch2 | APPLY — LangGraph conditional routing as the loop's implementation substrate | p. ~69, ~73 |
| Ch6 | APPLY — the four retrieval stages mapped onto the loop | p. ~178 |
| Ch8 | APPLY — four-phase data-analysis loop; V&V adds a **meta-reasoning layer over** it | p. ~235, ~244 |
| Ch10 | APPLY — SMPA (sense-model-plan-act) as the loop for content generation | p. ~325, ~330 |
| Ch11 | APPLY — SMPA structures all three perceptual pipelines; state estimation precedes reasoning precedes actuation | p. ~358 |
| Ch12 | APPLY — ethical checkpoint interposed between Reason and Act | p. ~361 |
| Ch15 | APPLY — loop mapped to the instructional cycle (Figure 15.1) | p. ~453, ~456 |
| Ch17 | APPLY — the substrate self-architecting agents learn to rewrite `(inferred)` | — |

### Memory (working / episodic / semantic)

| Ch | Role | Anchor |
|---|---|---|
| Ch5 | **INTRO** — three memory types + selection guide (trigger, query signal, failure mode if skipped); consolidation/summarization | p. ~167, ~171 |
| Ch3 | Pre-echo — memory as a prompt-architecture concern; what to remember lives in **context** | p. ~108 |
| Ch4 | APPLY — Memory Store as a microservice; memory-level threats; memory governance | p. ~134, ~137, ~140 |
| Ch7 | APPLY — working vs. long-term (episodic + semantic) in multi-agent shared memory | p. ~220 |
| Ch10 | APPLY — **dual-memory hierarchy (RAD loop)**: working = RAM, semantic = Disk; archival is an explicit operation | p. ~315, ~321 |
| Ch13 | APPLY — `ClinicalKnowledgeBase` dual-memory; case study maps all three | p. ~395, ~405 |
| Ch14 | APPLY — longitudinal client models on the same three stores | p. ~434 |
| Ch17 | Gap named — current agents lack the **consolidation** step (complementary learning systems) | p. ~528 |

### RAG / retrieval

| Ch | Role | Anchor |
|---|---|---|
| Ch6 | **INTRO** — four-stage retrieval architecture, chunking strategies, failure diagnosis, provenance in parallel | p. ~178–183 |
| Ch2 | Pre-req — vector DB selection, embedding models, embedding-consistency rule | p. ~81 |
| Ch3 | Positioning — few-shot **vs.** RAG (Table 3.2); both live in **context**; add RAG when data is private or fast-changing | p. ~95, ~111 |
| Ch5 | Pointer — re-ranking retrieved documents deferred to Ch6 | p. ~174 |
| Ch10 | APPLY — semantic memory retrieved by cosine similarity; researcher agent grounds via RAG | p. ~315, ~327 |
| Ch13 | APPLY — biomedical embeddings; `ProductionLiteratureScanner` reliability wrapper | p. ~397, ~407–409 |
| Ch14 | **EXTENDS** — hybrid dense+sparse with authority-weighted re-ranking (0.5/0.3/0.2); "demands architectures well beyond standard RAG" | p. ~439, ~441 |

### POMDP / belief state

| Ch | Role | Anchor |
|---|---|---|
| Ch13 | **INTRO** — clinical reasoning as POMDP; Bayesian belief update `P(s\|o) ∝ P(o\|s)·P(s)`; priors from prevalence, likelihoods from sensitivity/specificity | p. ~393, ~400–401 |
| Ch15 | **EXTENDS Ch13** — book states the education POMDP *extends the stochastic decision-making frameworks of Ch13*; decomposed into BKT + planner + scheduler | p. ~453–454 |
| Ch16 | Parallel use — continuous POMDP for physical environments; particle / extended Kalman filters at 10–100 Hz. **The book never links this back to Ch13 or Ch15** | p. ~493 |

> Correct chain: **Ch13 → Ch15 is declared. Ch16's POMDP is unlinked** — any Ch16↔Ch13/Ch15 statement is `(inferred)`.

### HITL / human supervision / escalation

| Ch | Role | Anchor |
|---|---|---|
| Ch5 | **INTRO** — `intelligent_escalation_decision()`, five weighted factors with thresholds; guided resolution via `execute_with_checkpoints()`; "not every problem can or should be solved autonomously" | p. ~152, ~160–161 |
| Ch4 | Governance framing — risk-based escalation; oversight must account for *human* cognitive biases | p. ~142 |
| Ch6 | APPLY — confidence-gated HITL at the validation stage; design for HITL from day one | p. ~186, ~191 |
| Ch7 | APPLY — HITL as a **planned first-class branch**, not a failure path; explicit "Step 2.5" gate | p. ~224, ~228 |
| Ch8 | APPLY — HITL triggers on low confidence in consistency analysis | p. ~244 |
| Ch9 | APPLY — risk-tiered checkpoint (low auto / medium review / high cross-functional) | p. ~296–302 |
| Ch11 | Field lesson — facility managers override what they don't understand; log *why* | p. ~357 |
| Ch13 | APPLY — `SafetyMonitor`, threshold **derived from cost asymmetry** (0.12–0.18 band → 0.15) | p. ~401, ~404 |
| Ch14 | APPLY — state-graph routing makes human checkpoints *enforceable*; force multipliers, not autonomous deciders | p. ~424, ~449 |
| Ch15 | APPLY — Slack alert to instructors on repeated failure / distress / suspected dishonesty | p. ~469 |
| Ch17 | Critique — **the supervisor trap**; collaboration spectrum replaces exception-handler framing | p. ~529 |

### Confidence, thresholds, calibration

| Ch | Role | Anchor |
|---|---|---|
| Ch5 | **INTRO of thresholded escalation** — confidence < 0.8, complexity > 0.7, business impact > 0.6; learning triggers at success_score < 0.7 | p. ~155, ~160 |
| Ch4 | Confidence-based escalation for model routing | p. ~129 |
| Ch6 | OCR `CONFIDENCE_THRESHOLD = 60` routes the whole downstream pipeline; 95% field accuracy / <8% human review | p. ~185, ~190–191 |
| Ch7 | Semantic-search cutoff ~0.7; conflict detection ~0.7; consensus ~95%; claim guard `confidence_score < 0.85` | p. ~211, ~224, ~231 |
| Ch8 | GPS `CONFIDENCE_THRESHOLD = 0.70`; rubric mean of three 0–1 criteria | p. ~258, ~260 |
| Ch13 | **INTRO of calibration proper** — Brier decomposition (Reliability / Resolution / Uncertainty); Platt or temperature scaling minimizes Reliability | p. ~401 |
| Ch14 | Composite risk 0.4 vol / 0.35 drawdown / 0.25 VaR; bands ≥7.0 HIGH, ≥4.0 MODERATE | p. ~431–432 |
| Ch15 | BKT posterior as a *running probability*, explicitly not a binary label; `min_consistency 0.7` style gating | p. ~462–465 |

### Security, threat model, zero trust

| Ch | Role | Anchor |
|---|---|---|
| Ch4 | **INTRO** — threat taxonomy by attack-surface layer (input / execution / memory), risk levels, zero trust for agents, defense-in-depth (5 layers), incident preparedness | p. ~137–140 |
| Ch5 | Pointer — prompt injection defenses and secure tool access tie to Ch4 | p. ~175 |
| Ch6 | Access control enforced **at the retrieval level**, not after | p. ~183 |
| Ch12 | **APPLY, declared** — three attack vectors against the ethical validator, drawn from Ch4's threat model | p. ~368 |
| Ch16 | Hardware emergency stop bypassing all software layers; `A_safe(s)` below the control interface | p. ~497 |

### Compliance gates

| Ch | Role | Anchor |
|---|---|---|
| Ch3 | Seed — **context** is the compliance guardrail; explicit prohibitions + mandatory escalation live there | p. ~119–120 |
| Ch4 | Regulatory layer — GDPR/CCPA/EU AI Act, NIST AI RMF, EO 14110; anticipatory compliance design | p. ~142–143 |
| Ch9 | **INTRO as an architecture** — Compliance-Driven agent; policy engine (OPA/Rego), "policies are the test suite for compliance"; hard vs. soft failure | p. ~286–295 |
| Ch12 | Value alignment hierarchy: safety > ethics > task; fairness pipeline for regulated decisions | p. ~362, ~368–369 |
| Ch13 | **Compliance by architecture, not convention**; 7-year signed append-only audit; provenance carries regulatory weight | p. ~397, ~405, ~419 |
| Ch14 | **Compliance-by-architecture named** — self-correcting `recommend → comply → revise → comply`; "structurally impossible for a non-compliant recommendation to reach the client" | p. ~436–437 |
| Ch16 | Conservative constraint fusion — any single unsatisfied *or stale* domain vetoes the envelope | p. ~519–520 |

### Multi-agent / chain-of-agents

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **INTRO of the ladder** — Five Levels of Agent Interaction, topping out at multi-agent | p. ~53 |
| Ch3 | Prompt-driven inter-agent protocols; PTCF encoded in the **message schema** | p. ~96, ~117–118 |
| Ch4 | Infrastructure — Kubernetes + Kafka; quadratic message growth; observability is *particularly* critical here | p. ~127, ~130–131 |
| Ch7 | **INTRO as an architecture** — Chain-of-Agents, cooperation protocol (4 pillars, 4 elements), arbitration in 4 stages | p. ~217–224 |
| Ch9 | APPLY — project manager + backend + frontend + tester over shared LangGraph state | p. ~280–284 |
| Ch10 | APPLY — researcher → writer → editor chain with feedback loop | p. ~326–327 |
| Ch13 | APPLY — `PatientDataPipeline` sub-agents **following Ch7's patterns** (declared) | p. ~399–400 |
| Ch14 | APPLY — supervised multi-agent with state-graph routing; RetailAdvisor's five specialists | p. ~424, ~437–438 |
| Ch15 | **EXTENDS** — Collective Intelligence: Condorcet, weighted voting, rotating adversarial critic, bounded rounds, emergence mechanisms | p. ~472–484 |
| Ch17 | **EXTENDS further** — agent *societies*: reputation ledgers, coalition formation, stigmergy, norm emergence | p. ~525–526 |

### Tool orchestration

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **INTRO of the concept** — tool use is the Act phase; Level 2 of the Progression Framework | p. ~36, ~63 |
| Ch3 | Prompts for tool-using agents must guide *what*, *how*, and *when* | p. ~104 |
| Ch7 | **INTRO as an architecture** — reasoning core + registry + execution engine + tool chest; selection funnel; six recovery strategies | p. ~206–216 |
| Ch9 | APPLY — test runners, file I/O, sandboxes as the agents' hands (declared, p. ~266) | p. ~266 |
| Ch10 | APPLY — `AssetRequest` contract; the text agent never calls the image API | p. ~332 |
| Ch11 | APPLY — declared as one of the chapter's two foundations | p. ~338 |
| Ch16 | APPLY — tool-mediated indirection so the LLM cannot bypass the safety layer | p. ~499 |

### Planning

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **INTRO** — Level 3 of the Progression Framework | p. ~63 |
| Ch3 | Task decomposition must be *explicitly embedded* in **task** + **format**; ToT walkthrough | p. ~106–107, ~114–116 |
| Ch5 | **INTRO as an architecture** — Planning agent, hierarchical decomposition, STRIPS/PDDL vs. LLM planning, Table 5.1 planning vs. decision-making | p. ~162–166 |
| Ch4 | Infrastructure — planning DAGs and checkpointing for deliberative agents | p. ~126–127 |
| Ch7 | Plans as tool-call sequences with `data_state` threaded through | p. ~216–217 |
| Ch9 | ToT decomposition into a dependency graph with topological ordering | p. ~280 |
| Ch15 | Curriculum planning as constrained shortest path on a DAG | p. ~454 |
| Ch16 | Task planning as the top band of the control hierarchy (0.1–1 Hz), PDDL or LLM | p. ~494 |

### MCP / A2A  ⚠️ conflicting attribution inside the book

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **ACTUAL INTRO** — MCP standardizes tool discovery/invocation; A2A defines inter-agent message passing | p. ~47, ~49–50 |
| Ch2 | Cloud platforms; **explicitly credits Ch1 with introducing them** (declared) | p. ~79–80, ~87, ~89 |
| Ch6 | APPLY — MCP removes hardcoded per-API logic for academic databases; A2A splits search from synthesis. Points to Ch8 for the deep dive | p. ~197–198 |
| Ch7 | APPLY — cooperation protocol = MCP/A2A at the wire level. **Says they were "introduced in Chapter 6"** — contradicts Ch2 | p. ~219 |
| Ch13 | APPLY — same split as Ch6, for literature databases | p. ~409 |

> **Full-text scan for `MCP` / `Model Context Protocol` / `A2A` / `Agent-to-Agent` returns pages 8, 47, 49, 50, 79, 87, 89, 197, 219, 409, 532 (index) — and nothing in Ch8.** Ch6's "see Chapter 8" pointer is a dangling forward reference. Route MCP questions to Ch1 (definition) → Ch6/Ch13 (retrieval use) → Ch7 (protocol mechanics). Never to Ch8.

### Explainability

| Ch | Role | Anchor |
|---|---|---|
| Ch4 | Seed — transparency beyond feature importance; longitudinal transparency; audience-tailored explanations | p. ~141 |
| Ch11 | Structured output parsing as an explainability mechanism (reasoning trace separated from answer) | p. ~347 |
| Ch12 | **INTRO as an architecture** — SHAP (global) + LIME (per-prediction); reasoning chain stored as audit artifact; one-size explanation is an anti-pattern | p. ~378, ~380 |
| Ch13 | APPLY — audience-adapted explanation (clinician / patient / auditor) from one diagnostic trace; **adoption tracks explainability, not accuracy** | p. ~395, ~404–405, ~419 |
| Ch14 | APPLY — immutable audit log as the compliance substrate; traceable data lineage | p. ~424, ~438 |
| Ch15 | APPLY — rejected consensus elements recorded with explanations, **supporting Ch12's requirements** (declared) | p. ~477 |
| Ch16 | Explainability follows from graph structure — trace the influence path through typed edges | p. ~506 |
| Ch17 | Peer audit protocol; behavioral drift detection (KS / Jensen-Shannon) | p. ~526 |

### Learning / adaptation

| Ch | Role | Anchor |
|---|---|---|
| Ch1 | **INTRO** — Level 4 learning agents, the top tier | p. ~63 |
| Ch5 | `learn_from_interaction()`; adjust autonomy thresholds when success_score < 0.7 | p. ~155 |
| Ch6 | Scientific Research agent placed at **Level 4** on the spectrum | p. ~199 |
| Ch8 | Meta-learning engine builds a reusable library of cognitive patterns | p. ~249 |
| Ch9 | **INTRO as an architecture** — Self-Improving agent closed control loop; drift detection; catastrophic forgetting risk | p. ~296–308 |
| Ch10 | Adaptive optimization cycle as an RL objective over content iterations | p. ~329 |
| Ch11 | Online learning to recalibrate drifting thermal models | p. ~357 |
| Ch13 | Experiment Tracker closes digital→physical loop — **"the mechanism that enables the Level 4 learning behavior introduced in Chapter 1"** (declared) | p. ~416 |
| Ch15 | BKT belief update + SM-2 as the runtime learning policy; expertise weights as a learning lever | p. ~463, ~467, ~480 |
| Ch17 | Self-architecting agents; **improvement velocity is the ROI metric that compounds** | p. ~524–525, ~529 |

### Cross-cutting quick refs

| Concept | INTRO | Key APPLY sites |
|---|---|---|
| **Agentic AI Progression Framework (L0–L4)** | Ch1 p. ~63 | Ch3 capability spectrum p. ~104–105; Ch6 Table 6.1 p. ~199; Ch13 p. ~416; Ch17 crawl/walk/run p. ~528 |
| **PTCF prompting** | Ch3 p. ~98–103 | Ch10 persona layer p. ~316; Ch5 LLM-powered cognition p. ~157 |
| **CoT / ToT** | Ch3 p. ~113–116 | Ch5 planning (defers to Ch3, declared p. ~163); Ch9 ToT decomposition p. ~280; Ch11 CoT for audio p. ~345 |
| **Provenance / audit trail** | Ch6 p. ~179, ~191 | Ch7 p. ~231; Ch8 p. ~243; Ch12 p. ~378; Ch13 p. ~396–397, ~405; Ch14 p. ~438 |
| **Observability** | Ch2 p. ~35 (+Ch1 p. ~40) | Ch4 p. ~139–140; Ch5 LangSmith p. ~174; Ch7 p. ~216; Ch13 telemetry p. ~409 |
| **Fairness / bias** | Ch4 p. ~142 | Ch12 impossibility theorem + three regimes p. ~368–369; Ch9 bias monitoring p. ~307 |
| **Guardrails / safety checks** | Ch5 `autonomous_safety_check` p. ~159–160 | Ch10 sentinel p. ~319–320; Ch12 ethical checkpoint p. ~361; Ch16 `A_safe(s)` p. ~497 |

---

## 2. Prerequisite chains

### Declared chains — "read X before Y because Y assumes it"

Every dependency below is stated by the book. Wording and full page lists are in §4's master table; this is the ordering view.

| Y (target) | assumes | Anchor in Y |
|---|---|---|
| Ch2 | Ch1 | p. ~69, ~73, ~78, ~79, ~80 |
| Ch4 | Ch1, Ch3 | p. ~124, ~146 |
| Ch5 | Ch1, Ch2, Ch3 | p. ~149–161, ~174, ~152/~163 |
| Ch7 | Ch6 | p. ~219 |
| Ch8 | Ch5 | p. ~237, ~244 |
| Ch9 | Ch5, Ch7 | p. ~266 |
| Ch10 | Ch1, Ch5 | p. ~325/~328, ~313/~314 |
| Ch11 | Ch1, Ch7 | p. ~338, ~345 |
| Ch12 | Ch1, Ch4, Ch8 | p. ~361/~363, ~368, ~378 |
| Ch13 | Ch1, Ch5, Ch6, Ch7 | p. ~416, ~405, ~406, ~399–400 |
| Ch14 | Ch5, Ch12 | p. ~434, ~422/~432/~436/~437 |
| Ch15 | Ch1, Ch12, Ch13 | p. ~453/~456, ~477, ~453 |
| Ch16, Ch17 | *(nothing declared)* | — |

Forward-declared (X announces Y): Ch1→Ch2 `[pos]` ~67 · Ch2→Ch3 `[pos]` ~91 · Ch3→Ch4 `[num]` ~122 · Ch4→Ch5 `[num]` ~146 · Ch5→Ch6 `[num]` ~174 + `[pos]` ~176 · Ch5→Ch7 `[num]` ~173–174 · Ch5→Ch4 `[num]` ~174–175 · Ch6→Ch7 `[pos]` ~201 · Ch6→Ch8 `[num]` ~198 ⚠️ · Ch7→Ch8 `[num]` ~232 · Ch7→Ch9 `[pos]` ~232 · Ch8→Ch9 `[num]` ~264 · Ch9→Ch10 `[pos]` ~309–310 · Ch10→Ch11 `[pos]` ~337 · Ch10→Ch14 `[num]` ~329 ⚠️ · Ch11→Ch12 `[num]` ~358 · Ch12→Ch13 `[pos]` ~391 · Ch13→Ch14 `[pos]` ~420 · Ch14→Ch15 `[pos]` ~450 · Ch15→Ch16 `[pos]` ~485.

### Inferred chains

| Chain | Why | Status |
|---|---|---|
| Ch2 → Ch6 | LlamaIndex / vector DBs are the retrieval toolkit | `(inferred)` |
| Ch2 → Ch7, Ch2 → Ch9 | LangGraph / CrewAI / AutoGen are the orchestration substrate | `(inferred)` |
| Ch3 → Ch7, Ch3 → Ch10 | inter-agent message schemas; persona layer sits on PTCF | `(inferred)` |
| Ch4 → Ch7, Ch4 → Ch9 | HITL gates and CI/CD are deployment concerns | `(inferred)` |
| Ch11 → Ch16 | multi-modal perception feeds the belief-state update | `(inferred)` — Ch16 states nothing |
| Ch13/Ch15 → Ch16 | shared POMDP formalism | `(inferred)` — Ch16 states nothing |
| Ch9/Ch12 → Ch17 | self-improvement seeds self-architecting; ethics becomes internalized governance | `(inferred)` — Ch17 states nothing |

### Minimum reading routes

| Target | Route | Skippable |
|---|---|---|
| **Ch5** cognitive architectures | Ch1 → Ch3 → Ch5 | Ch2, Ch4 |
| **Ch6** retrieval | Ch1 → Ch2 (vector DBs) → Ch5 → Ch6 | Ch3, Ch4 |
| **Ch7** orchestration | Ch1 → Ch5 → Ch6 (MCP/A2A) → Ch7 | Ch2–Ch4 |
| **Ch8** data reasoning | Ch1 → Ch5 → Ch7 → Ch8 | Ch2–Ch4, Ch6 |
| **Ch9** software dev | Ch1 → Ch5 → Ch7 → Ch9 (Ch8 helps) | Ch2–Ch4, Ch6 |
| **Ch10** conversational/content | Ch1 → Ch3 → Ch5 → Ch10 | Ch2, Ch4, Ch6–Ch9 |
| **Ch11** multi-modal | Ch1 → Ch7 → Ch11 | everything else |
| **Ch12** ethics/XAI | Ch1 → Ch4 (threat model) → Ch8 (V&V) → Ch12 | Ch2, Ch3, Ch5–Ch7 |
| **Ch13** healthcare/science | Ch1 → Ch5 (memory) → Ch6 (research agent) → Ch7 → Ch13 | Ch2–Ch4, Ch8–Ch12 |
| **Ch14** finance/legal | Ch5 (memory) → Ch12 → Ch14 | Ch1–Ch4, Ch6–Ch11, Ch13 |
| **Ch15** education/collective | Ch1 → Ch13 (POMDP) → Ch12 → Ch15 | Ch2–Ch11, Ch14 |
| **Ch16** embodied | Ch1 → Ch11 → Ch16 `(inferred route — Ch16 declares no prerequisites)` | — |
| **Ch17** epilogue | Ch9 → Ch12 → Ch16 `(inferred route — Ch17 declares no prerequisites)` | — |

**Spine** (declared, unbroken): Ch1 → Ch3 → Ch4 → Ch5 → Ch6 → Ch7 → Ch8 → Ch9. Everything from Ch10 on branches off Ch5 plus one or two domain-specific gates (Ch12 for regulated domains, Ch13 for POMDP domains).

---

## 3. Framework combinations

Marked `(inferred)` where the composition is evident from two verified pieces but the book does not name the pairing.

| Combination | Purpose | Basis |
|---|---|---|
| **PTCF + RAG** | Start with a PTCF prompt; add RAG when the agent needs private or fast-changing data; reach for LoRA/QLoRA only after prompting plateaus | Declared (Ch 3, p. ~95) |
| **PTCF + few-shot** | Few-shot examples live inside the **context** pillar — a lightweight alternative or complement to RAG | Declared (Ch 3, p. ~108, ~111) |
| **PTCF + inter-agent schema** | Encode persona/task/context/format in the message envelope, not only in each agent's system prompt | Declared (Ch 3, p. ~96, ~117–118) |
| **Cognitive loop + safety gate** | `autonomous_safety_check` between planning and execution; escalation score over five factors | Declared (Ch 5, p. ~159–161) |
| **Cognitive loop + ethical checkpoint** | Perceive → Reason → Plan → **[ETHICAL CHECKPOINT]** → Execute | Declared (Ch 12, p. ~361) |
| **Cognitive loop + meta-reasoning layer** | Primary agent solves; V&V layer examines *how* the solution was produced | Declared (Ch 8, p. ~244) |
| **Chain-of-agents + compliance gate** | Enforcement at the **supervisor**, never inside specialists; `recommend → comply → revise → comply` | Declared (Ch 14, p. ~432, ~436–437) |
| **Chain-of-agents + arbitration** | Conflict detection → automated arbitration → confidence consensus → human escalation | Declared (Ch 7, p. ~223–224) |
| **Chain-of-agents + shared memory** | Cooperation protocol only works with an authoritative shared workspace (working + episodic + semantic) | Declared (Ch 7, p. ~218, ~220) |
| **Chain-of-agents + rotating adversarial critic** | Fights correlated error so Condorcet's guarantee approximately holds; +12% on risk identification | Declared (Ch 15, p. ~472, ~477) |
| **POMDP + knowledge tracing (BKT)** | BKT *is* the belief-state estimator in the decomposed education POMDP; planner = action selection; SM-2 = temporal planning | Declared (Ch 15, p. ~454) |
| **POMDP + IRT cold start** | IRT placement produces the initial belief state that seeds curriculum, hints, pacing | Declared (Ch 15, p. ~459–460) |
| **POMDP + calibration** | Belief state is only usable if confidence is calibrated — Brier Reliability minimized by Platt/temperature scaling | Declared (Ch 13, p. ~401) |
| **POMDP + particle filter + deterministic controller** | Asymmetric control loop: probabilistic reasoning above, deterministic control below | Declared (Ch 16, p. ~489, ~493) |
| **RAG + provenance + audience-adapted explanation** | One traceable path yields clinician report, patient summary, and regulator audit record | Declared (Ch 13, p. ~395, ~404–405) |
| **Hybrid RAG + authority re-ranking + citation gate** | Legal retrieval: dense+sparse, 0.5/0.3/0.2 re-rank, then verification against an authoritative KB | Declared (Ch 14, p. ~439–443) |
| **RAG + confidence-gated HITL** | Confidence score routes automatic integration vs. human review at the validation stage | Declared (Ch 6, p. ~186) |
| **TDG + compliance loop** | Functional-correctness loop and permissibility loop run in the same pipeline — orthogonal concerns | Declared (Ch 9, p. ~286) |
| **Tool registry + failure memory + fallback chain** | Circuit-breaker status field in the registry drives constraint filtering and graceful degradation | Declared (Ch 7, p. ~207, ~215) |
| **Persona stack + safety sentinel** | Deterministic circuit breaker upstream; generation is downstream of policy | Declared (Ch 10, p. ~319–320) |
| **Brand CSP + editor-agent feedback loop** | Hard constraints scored `C = 1/n · Σ φ(Aᵢ, G)`, looped until pass | Declared (Ch 10, p. ~325–327, ~331) |
| **Knowledge graph + influence propagation + governance controls** | Clamping, expert override, audit logging, rollback on LLM-estimated coupling weights | Declared (Ch 16, p. ~505–508) |
| **PTCF + tiered model routing** | Cheap model for Level-1 prompts, escalate on low confidence — composes Ch3's capability spectrum with Ch4's cost framework | `(inferred)` |
| **RAG + threat model** | Retrieval-level access control (Ch6 p. ~183) is Ch4's memory-level mitigation applied to the retrieval surface | `(inferred)` |
| **Chain-of-agents + SHAP/LIME** | Per-specialist attribution for a supervised multi-agent recommendation | `(inferred)` |
| **Self-improving loop + ethical circuit breaker** | Ch9's risk-tiered HITL is the precursor to Ch17's graduated response cascade | `(inferred)` |
| **Multi-modal perception + belief state** | Ch11's sensor pipelines are the observation model for Ch16's particle filter | `(inferred)` |
| **Memory consolidation + episodic store** | Ch17 names the missing batch-replay step for Ch5's episodic memory | `(inferred)` — Ch17 cites no chapter |

---

## 4. Declared cross-references by chapter — and the verdict on each current "Connects To"

Method: full-text extraction of every `Chapter N` token across all 310 EPUB splits, attributed to the containing chapter by page anchor, with running headers discarded and every hit read in context. Positional transitions ("in the next chapter…") were collected separately.

**Legend** — `[num]` declared by number · `[pos]` declared by position, no number · `KEEP` unchanged · `FIX` wrong target/page/strength · `DEMOTE` real but must be marked `(inferred)` · `ADD` missing declared link.

### Master table

| Ch | Declared targets | Page(s) | Direction | Nature |
|---|---|---|---|---|
| **Ch1** | *(none numbered)* — Ch2 `[pos]` | ~67 | forward | "the following chapter provides a practical toolkit" |
| **Ch2** | Ch1 `[num]` ×5 | ~69, ~73, ~78, ~79, ~80 | back | cognitive loop; LangGraph routing; cognition core; multi-agent; MCP/A2A |
| | Ch3 `[pos]` | ~91 | forward | "advanced prompt engineering" |
| **Ch3** | Ch4 `[num]` + `[pos]` | ~121, ~122 | forward | scaling, security, failure/recovery, governance |
| **Ch4** | Ch1 `[num]` | ~124 | back | extends the ADL's deployment phase |
| | Ch3 `[num]` | ~146 | back | "In Chapter 3, we explored the art of agent prompting" |
| | Ch5 `[num]` | ~146 | forward | "Chapter 5 will present the cognitive foundational architecture" |
| **Ch5** | Ch1 `[num]` ×10 | ~149, ~150, ~151, ~153, ~154, ~155, ~156, ~157, ~159, ~161 | back | loop, perception, reasoning, action, learning, design considerations, case study |
| | Ch2 `[num]` | ~174 | back | LLM selection, fine-tuning, model scaling |
| | Ch3 `[num]` ×2 | ~152, ~163 | back | prompts shape cognition; CoT prerequisite for LLM planning |
| | Ch4 `[num]` ×2 | ~174, ~175 | forward | testing/continuous improvement; prompt injection + secure tool access |
| | Ch6 `[num]` | ~174 | forward | re-ranking retrieved documents |
| | Ch7 `[num]` ×2 | ~173, ~174 | forward | multi-agent workflows; tool-integration error handling |
| | Ch6 `[pos]` | ~176 | forward | "we turn to Information Retrieval and Knowledge agents" |
| **Ch6** | Ch8 `[num]` | ~198 | forward | "For a deeper understanding of MCP, see Chapter 8" — **⚠️ dangling, Ch8 has no MCP** |
| | "later chapters" | ~179–180 | forward | deployment and scaling patterns |
| | Ch7 `[pos]` | ~201 | forward | implementation strategies, enterprise scale |
| **Ch7** | Ch6 `[num]` ×2 | ~219 | back | MCP/A2A formalize the cooperation protocol |
| | Ch8 `[num]` | ~232 | forward | orchestration patterns applied to data analysis |
| | Ch9 `[pos]` | ~232 | forward | "Later chapters introduce software development agents" |
| **Ch8** | Ch5 `[num]` ×2 | ~237, ~244 | back | Data Analysis extends perception-reasoning-action; V&V adds meta-reasoning |
| | Ch9 `[num]` | ~264 | forward | cognitive loop, verification pipeline, meta-learning carry into coding agents |
| **Ch9** | Ch5 `[num]`, Ch7 `[num]` | ~266 | back | "core cognitive architectures from Chapter 5 and the tool-use frameworks from Chapter 7" |
| | Ch10 `[pos]` | ~309–310 | forward | conversational and content creation agents |
| **Ch10** | Ch1 `[num]` ×2 | ~325, ~328 | back | SMPA paradigm; contrast with deterministic if-then marketing automation |
| | Ch5 `[num]` ×2 | ~313, ~314 | back | specialization of memory-augmented + planning agents; hybrid patterns |
| | Ch14 `[num]` | ~329 | forward | "longitudinal learning capability is explored in Chapter 14" — **⚠️ Ch14 covers finance/legal, not long-term learning/fine-tuning** |
| | Ch11 `[pos]` | ~337 | forward | agents that perceive the physical world |
| **Ch11** | Ch1 `[num]` ×2 | ~338, ~345 | back | cognitive architecture; deliberate reasoning precedes action |
| | Ch7 `[num]` | ~338 | back | tool orchestration patterns |
| | Ch12 `[num]` | ~358 | forward | explaining decisions, bias, accountability |
| **Ch12** | Ch1 `[num]` ×2 | ~361, ~363 | back | ethical checkpoint extends the loop; deontic operators inside the loop |
| | Ch4 `[num]` | ~368 | back | three attack vectors from Ch4's threat model |
| | Ch8 `[num]` | ~378 | back | V&V agent — every verification step must be traceable |
| | Ch13 `[pos]` | ~391 | forward | healthcare and scientific discovery |
| **Ch13** | Ch5 `[num]` | ~405 | back | episodic / semantic / working memory patterns |
| | Ch6 `[num]` | ~406 | back | Scientific Discovery builds on Ch6's research agent |
| | Ch7 `[num]` | ~399–400 | back | multi-agent patterns for `PatientDataPipeline` |
| | Ch1 `[num]` | ~416 | back | Level 4 learning behavior |
| | Ch14 `[pos]` | ~420 | forward | financial and legal, comparable regulatory/auditability constraints |
| **Ch14** | Ch12 `[num]` ×4 | ~422, ~432, ~436, ~437 | back | foundations; compliance agent architecture; ethical reasoning at data-access layer; compliance registry pattern |
| | Ch5 `[num]` | ~434 | back | working / episodic / semantic for longitudinal client models |
| | Ch15 `[pos]` | ~450 | forward | education and knowledge agents |
| **Ch15** | Ch1 `[num]` ×2 | ~453, ~456 | back | cognitive loop mapped to the instructional cycle |
| | Ch13 `[num]` | ~453 | back | POMDP extends Ch13's stochastic decision frameworks |
| | Ch12 `[num]` | ~477 | back | explainability requirements |
| | Ch16 `[pos]` | ~485 | forward | embodied agents bridging digital and physical |
| **Ch16** | **none** | — | — | only "Earlier chapters explored agents operating in purely digital environments" (p. ~488), unattributed |
| **Ch17** | **none** | — | — | only "Early chapters established the core abstractions" (p. ~521–522) and "beyond earlier chapters" (p. ~525) |

### Per-chapter verdict on the current "Connects To"

| Ch | Current entry | Verdict | Action for phase 2 |
|---|---|---|---|
| **Ch1** | Ch5 — cognitive architectures implement the loop | **DEMOTE** | mark `(inferred)`; Ch1 declares nothing |
| | Ch4 — deployment per progression stage | **DEMOTE** | mark `(inferred)` |
| | Ch7 — tool use is the Act phase | **DEMOTE** | mark `(inferred)` |
| | — | **ADD** | Ch2 `[pos]` (Ch 1, p. ~67) — the toolkit chapter |
| **Ch2** | Ch6 — LlamaIndex for retrieval | **DEMOTE** | mark `(inferred)` |
| | Ch7 — LangGraph for orchestration | **DEMOTE** | mark `(inferred)` |
| | Ch9 — CrewAI/AutoGen for software multi-agent | **DEMOTE** | mark `(inferred)` |
| | — | **ADD** | Ch1 `[num]` ×5 (p. ~69, ~73, ~78, ~79, ~80) — the only declared links, currently absent |
| | — | **ADD** | Ch3 `[pos]` (p. ~91) |
| **Ch3** | Ch4 (p. ~122) | **KEEP** | correct as written |
| | Ch1 — capability spectrum = Progression Framework | **DEMOTE** | conceptually sound, not declared in Ch3 |
| | Ch2 — LangChain/CrewAI/AutoGen; RAG alternative | **DEMOTE** | mark `(inferred)` |
| | Ch7 — inter-agent protocols extend into orchestration | **DEMOTE** | mark `(inferred)`; note the reverse is also undeclared |
| **Ch4** | Ch1 (p. ~124) | **KEEP** | verified |
| | Ch3 (p. ~146) | **KEEP** | verified |
| | Ch5 (p. ~146) | **KEEP** | verified |
| | | | **Ch4 is clean — no change required.** |
| **Ch5** | Ch1 (p. ~149, ~155) | **KEEP + enrich** | ten anchors exist: ~149, ~150, ~151, ~153, ~154, ~155, ~156, ~157, ~159, ~161 |
| | Ch2 (p. ~174) | **KEEP** | verified |
| | Ch3 (p. ~163) | **KEEP + enrich** | add p. ~152 (prompts shape cognitive behavior) |
| | Ch4 (p. ~174–175) | **KEEP** | verified |
| | Ch6 (p. ~174) | **KEEP** | verified |
| | Ch7 (p. ~173, ~174) | **KEEP** | verified |
| | — | **ADD** | Ch6 `[pos]` (p. ~176) — summary hand-off |
| **Ch6** | Ch8 (p. ~198) | **KEEP + flag** | declared, but Ch8 contains **no MCP content** — annotate as a dangling pointer and route MCP to Ch1/Ch7 |
| | Ch1 (p. ~177, ~199) | **DEMOTE** | Progression-Framework placement is real; the chapter never says "Chapter 1" |
| | Ch2 (p. ~180) | **FIX** | not a Ch2 reference. p. ~179–180 says "deployment and scaling patterns covered in **later chapters**". Retarget or demote |
| | Ch5 (p. ~176) | **FIX** | p. ~176 text is **Ch5's own summary** spilling across the split boundary — not a Ch6→Ch5 reference. Demote to `(inferred)` |
| | "Later chapters" MLOps (p. ~191) | **KEEP** | genuine unattributed forward reference |
| | — | **ADD** | Ch7 `[pos]` (p. ~201) |
| **Ch7** | Ch6 (p. ~219) | **KEEP** | verified ×2 |
| | Ch8 (p. ~232) | **KEEP** | verified `[num]` |
| | Ch9 "explicit forward reference" (p. ~232) | **FIX** | real but **unnumbered** — "Later chapters introduce software development agents". Downgrade to `[pos]` |
| | Ch5 `(inferred)`, Ch4 `(inferred)` | **KEEP** | already correctly marked — use this chapter as the formatting model |
| **Ch8** | Ch5 (p. ~243, ~247) | **FIX** | verified anchors are **p. ~237** (Data Analysis extends perception-reasoning-action — currently missing) and **p. ~244** (V&V meta-reasoning). The GPS↔Ch5-planning claim at ~247 is not declared → `(inferred)` |
| | Ch7 (p. ~234, ~248) | **DEMOTE** | no numbered Ch7 reference anywhere in Ch8 |
| | Ch9 (p. ~263) | **FIX page** | verified at **p. ~264** |
| | Ch12 — verification ↔ explainability | **DEMOTE** | not declared in Ch8. Note the **reverse is declared**: Ch12 p. ~378 cites Ch8's V&V agent |
| **Ch9** | Ch5 — cognitive architectures | **KEEP + cite** | add `[num]` (p. ~266) |
| | Ch7 — tool-use frameworks | **KEEP + cite** | add `[num]` (p. ~266) |
| | Ch12 — governance for self-improving agents | **DEMOTE** | mark `(inferred)` |
| | Ch17 — seed of self-evolving agents | **DEMOTE** | mark `(inferred)` |
| | — | **ADD** | Ch10 `[pos]` (p. ~309–310) |
| **Ch10** | Ch1 — SMPA; if-then contrast | **KEEP + cite** | `[num]` p. ~325 and p. ~328 |
| | Ch5 — specialization of memory-augmented/planning | **KEEP + cite** | `[num]` p. ~313, ~314 |
| | Ch14 — longitudinal learning and fine-tuning | **KEEP + flag** | genuinely declared `[num]` at **p. ~329**, but Ch14 delivers financial/legal agents, **not** long-term learning/fine-tuning. Annotate as an unfulfilled forward reference |
| | Ch3 — persona builds on PTCF | **DEMOTE** | mark `(inferred)` |
| | Ch7 — researcher→writer→editor is CoA | **DEMOTE** | mark `(inferred)` |
| | — | **ADD** | Ch11 `[pos]` (p. ~337) |
| **Ch11** | Ch1 (p. ~338, ~345) | **KEEP** | verified |
| | Ch7 (p. ~338) | **KEEP** | verified |
| | Ch12 (p. ~358) | **KEEP** | verified |
| | | | **Ch11 is clean — no change required.** |
| **Ch12** | Ch5 — checkpoint inserted into Ch5's loop | **FIX → Ch1** | misattribution. The book says the ethical reasoning agent "draws on the cognitive loop introduced in **Chapter 1**" (p. ~361), reinforced at p. ~363 |
| | Ch4 — extends responsible development | **KEEP + sharpen** | declared at **p. ~368**, specifically about the **threat model** and three attack vectors against ethical validators |
| | Ch13–16 — ethics prerequisite for regulated domains | **FIX** | only **Ch13** is declared, and only `[pos]` (p. ~391). Ch14/Ch15 are **incoming** links (Ch14 p. ~422/~432/~436/~437; Ch15 p. ~477); Ch16 has no relation in either direction |
| | — | **ADD** | Ch8 `[num]` (p. ~378) — V&V traceability. Currently missing entirely |
| **Ch13** | Ch7 (p. ~399) | **KEEP** | verified; anchor spans p. ~399–400 |
| | Ch5 (p. ~405) | **KEEP** | verified |
| | Ch6 (p. ~406–407) | **KEEP** | verified at p. ~406 |
| | Ch1 Level 4 (p. ~416) | **KEEP** | verified |
| | Ch14 forward (p. ~420) | **KEEP + mark** | correct, but `[pos]` — the transition names no chapter number |
| **Ch14** | Ch12 (p. ~422, ~432, ~436, ~437) | **KEEP** | all four verified exactly |
| | Ch5 (p. ~434) | **KEEP** | verified |
| | Ch13 (p. ~419) | **KEEP** | correctly described as the **incoming** link from Ch13's close (p. ~419–420) |
| | Ch15 (p. ~450) | **KEEP + mark** | `[pos]` — "In the next chapter, we explore education and knowledge agents" |
| | | | **Ch14 is clean — only the `[pos]` markers need adding.** |
| **Ch15** | Ch1 (p. ~453, ~456) | **KEEP** | verified |
| | Ch13 (p. ~453) | **KEEP** | verified — the single most load-bearing POMDP link in the book |
| | Ch12 (p. ~477) | **KEEP** | verified |
| | Ch16 (p. ~485) | **KEEP + mark** | `[pos]` |
| | | | **Ch15 is clean.** |
| **Ch16** | Ch5 — POMDP as physical extension of memory-augmented architecture | **FIX + DEMOTE** | not declared **and** conceptually wrong — POMDP is not an extension of memory-augmented architecture. Replace or drop |
| | Ch15 — same POMDP formulation | **DEMOTE** | `(inferred)`. The declared POMDP lineage is Ch13 → Ch15; Ch16 is outside it |
| | Ch11 — multi-modal perception feeds belief state | **DEMOTE** | `(inferred)`; strongest of the three, keep it but mark it |
| | — | **NOTE** | Ch16 is reached only by Ch15's `[pos]` hand-off (p. ~485). Ch16 itself cites nothing but "Earlier chapters" (p. ~488) |
| **Ch17** | Ch5 — loop as substrate; consolidation gap | **DEMOTE** | `(inferred)` |
| | Ch9 — execute-observe-learn-adapt seed | **DEMOTE** | `(inferred)` |
| | Ch12 — ethics becomes self-regulation | **DEMOTE** | `(inferred)` |
| | Ch16 — hierarchical control extends to nano/planetary | **DEMOTE** | `(inferred)` |
| | — | **NOTE** | Ch17 cites no chapter by number anywhere. All four entries are inferred; the chapter's only backward gesture is "Early chapters established the core abstractions" (p. ~521–522) |

### Phase-2 summary counts

| Verdict | Count |
|---|---|
| KEEP as-is | 24 |
| KEEP with page enrichment or `[pos]` marker | 11 |
| FIX (wrong target, page, or strength) | 8 |
| DEMOTE to `(inferred)` | 24 |
| ADD (declared link currently missing) | 10 |

**Chapters requiring no substantive change**: Ch4, Ch11, Ch14, Ch15.
**Chapters requiring the heaviest rewrite**: Ch16 and Ch17 (100% inferred), Ch2 (all declared links missing), Ch12 (one misattribution + one missing link + one overstated range), Ch6 (two false page attributions).
