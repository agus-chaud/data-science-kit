# Chapter 5: Foundational Cognitive Architectures

## Core Idea
Three cognitive architectures form the structural backbone of modern agent systems, each simulating one essential aspect of human cognition: the **Autonomous Decision-Making agent** (decision-making), the **Planning agent** (planning), and the **Memory-Augmented agent** (memory) (Ch 5, p. ~148). They are not mutually exclusive — the chapter's explicit claim is that they are complementary building blocks whose true potential emerges through strategic combination into integrated systems (Ch 5, p. ~171, Figure 5.4 on p. ~173). Together they realize the cognitive loop of perception, reasoning, action, and learning introduced in Ch 1 (Ch 5, p. ~175). Chapter epigraph: "Information is the resolution of uncertainty." — Claude Shannon (Ch 5, p. ~148).

The Autonomous Decision-Making agent is presented as the most fundamental building block, "the cornerstone upon which more advanced cognitive capabilities are constructed" (Ch 5, p. ~149). Each subsequent architecture layers onto it: planning adds multi-step decomposition over extended horizons; memory adds continuity across sessions.

## Frameworks Introduced

- **Autonomous Decision-Making agent** — agents that perceive their environment, evaluate possible actions against goals and constraints, choose, and execute *without continuous human supervision*, learning from outcomes by updating state or memories that inform the next loop (Ch 5, p. ~149). Running example throughout: a customer-support agent handling "My internet is down, and I can't connect."
  - **The cognitive loop in code** (`AutonomousCustomerServiceAgent.cognitive_loop`, Ch 5, p. ~155): (1) enhanced perception → (2) autonomous reasoning → (3) strategic action execution → (4) learning and adaptation → (5) update session memory for continuity. Instance state: `session_memory`, `decision_history`, `performance_metrics` (Ch 5, p. ~154).
  - **Perception: from raw input to structured intelligence** (Ch 5, p. ~150). Base `perceive_input()` returns message, timestamp, user_id, session_state, sentiment. `enhanced_perception()` layers *environmental awareness* on top: `current_load` (active sessions), `time_of_day`, `user_tier`, `recent_issues` (active system alerts), `agent_availability` (human agent status) (Ch 5, p. ~151). The point: the agent does not process the message in isolation — system load, time of day, user priority, and current network issues all shape the response strategy. A sophisticated perception layer also does validation, noise filtering, and preliminary structuring; its breadth and accuracy "fundamentally shape the agent's situational awareness" (Ch 5, p. ~151).
  - **Cognition: strategic reasoning** (Ch 5, p. ~151). `autonomous_reasoning()` builds a `decision_context` with intent, priority, `autonomy_level` (from user_tier + current_load + agent_availability), and `escalation_threshold` (from time_of_day + intent + priority).
  - **Strategy selection is a weighted decision tree**, scoring three candidate strategies across four axes each scored 0–1 (Ch 5, p. ~152):
    ```
    full_autonomous_resolution = autonomy_level_score*0.5 + (1-escalation_threshold)*0.3 + (1-complexity_score)*0.2
    immediate_escalation       = urgency_score*0.4 + (not agent_availability)*0.4 + escalation_threshold*0.2
    guided_autonomous_resolution = 0.5   # baseline fallback
    ```
    Highest composite score wins; ties favour guided resolution. These three strategies are the chapter's actual branching taxonomy — full autonomy, immediate escalation, or guided (checkpointed) resolution.
  - **Action: autonomous execution and adaptation** (Ch 5, p. ~153). `autonomous_action_execution()` dispatches on strategy: full autonomy runs `execute_autonomous_workflow()`; immediate escalation builds an escalation package and hands off to a human; guided resolution uses `execute_with_checkpoints()` for human oversight. It then closes the loop with `analyze_execution_results()` → `update_decision_models()`.
  - **Plans are dependency-aware task DAGs**, not linear lists. `create_autonomous_plan()` emits nodes `{"id", "action", "depends_on": [ids]}`; tasks with empty `depends_on` run immediately, downstream tasks wait for all listed ids. The billing_issue plan is 8 nodes (T1 fetch_account_details → … → T8 schedule_follow_up, with T7 joining T5 and T6); service_outage is 6 nodes; the default generic workflow is 4 (Ch 5, p. ~153–154).
  - **Learning** (`learn_from_interaction`, Ch 5, p. ~155): compute `success_score`, update user preferences, and **if `success_score < 0.7`** adjust autonomy thresholds and flag for model improvement; update strategy effectiveness; append the full record (perception, decision, outcome, timestamp) to `decision_history`.

- **LLM-powered cognition** (Ch 5, p. ~156–159). LangChain and OpenAI's Functions API are named as the orchestration layer for production deployment (Ch 5, p. ~157). The `LLMPoweredCognition` class returns a structured reasoning object: `intent`, `confidence`, `reasoning_chain`, `recommended_strategy`, `tool_requirements`, `escalation_assessment` (Ch 5, p. ~157).
  - **Perception with LLMs** splits into two functions: **input interpretation** (parsing unstructured natural language, logs, sensor readings into structured representations — "My internet is down" is interpreted as a *service outage* category for triage) and **contextual framing** (noise reduction, input-quality validation, anchoring data in conversation or prior interaction history) (Ch 5, p. ~157–158).
  - **Cognition with LLMs**: prompt templates for guided reasoning ("You are a customer service agent. Classify the user's intent and suggest the next best action."), complex reasoning and inference, and dynamic decision-making. Named extension techniques: **prompt chaining, scratchpad memory, and external tool integration**, which let LLM agents simulate aspects of traditional symbolic reasoning (Ch 5, p. ~159).
  - **Action with LLMs**: the LLM identifies and invokes tools directly through APIs or plugins — retrieving from a CRM, submitting a service request, generating a report (Ch 5, p. ~159).

- **Design considerations: guardrails and escalation** (Ch 5, p. ~159–161). Two safeguards are specified concretely.
  - `autonomous_safety_check(decision_context, planned_actions)` returns `(ok, violations)` and checks three classes (Ch 5, p. ~159–160): **financial impact limits** (`credit_amount > get_autonomous_credit_limit(user_tier)` → `credit_limit_exceeded`), **data access permissions** (`requires_sensitive_data_access` without `verify_autonomous_data_permissions` → `insufficient_data_permissions`), and **impact assessment** (`assess_customer_impact_risk(planned_actions) > escalation_threshold` → `high_impact_risk`). Guardrails against hallucination and prompt injection require strict input validation, output filtering, and prompt injection defenses (Ch 5, p. ~160).
  - `intelligent_escalation_decision()` computes a weighted escalation score over five explicit factors with book-specified thresholds (Ch 5, p. ~160): `safety_violations` (any), `confidence < 0.8`, `assess_issue_complexity > 0.7`, `check_escalation_preference(user_id)`, `calculate_business_impact > 0.6`. If `escalation_score > decision_context["escalation_threshold"]`, escalate. The rationale: agents must gracefully defer to a human on ambiguous situations, ethical dilemmas, or tasks beyond current capability (Ch 5, p. ~161).

- **Planning agent: orchestrating dynamic workflows with adaptive intelligence** (Ch 5, p. ~162). Where autonomous agents answer *what to do next*, Planning agents break complex goals into actionable sequences executed over extended periods — days, weeks, or months (Ch 5, p. ~162–163). The framing contrast: answering a single customer service query versus orchestrating a complete product launch.
  - **Five interconnected capabilities** a Planning agent must master (Ch 5, p. ~162): decompose high-level goals into actionable subtasks, sequence tasks appropriately, allocate resources efficiently, monitor progress continuously, and adapt plans dynamically when circumstances change.
  - **Hierarchical decomposition** is named "the most powerful strategy for Planning agents" — transforming abstract goals into structured trees of increasingly smaller, more concrete sub-goals (Ch 5, p. ~163).
  - **Two complementary planning paradigms** (Ch 5, p. ~163):
    - **Symbolic planning foundations** — **STRIPS** (STanford Research Institute Problem Solver) and **PDDL** (Planning Domain Definition Language) define states, actions, and goals in formal languages, enabling systematic search for optimal action sequences. Offers precision and optimality guarantees but requires comprehensive pre-modeling of the environment; strong in well-understood constrained domains, weak on novel or rapidly changing situations.
    - **LLM-powered dynamic planning** — **Tree-of-Thought (ToT)** and **Self-Ask** are named here as the techniques that dynamically generate and explore solution paths or investigative sub-questions, removing the need for rigid pre-programming. The chapter defers the actual mechanics to Ch 3, *The Art of Agent Prompting*: mastering prompt engineering for chain-of-thought is prerequisite. ToT is **not** an architecture introduced in this chapter — it is a one-paragraph pointer.
  - **Planning agent architecture** (Figure 5.2, marketing-campaign example, Ch 5, p. ~163–165). At the center sits the **Planning Agent Core** as strategic orchestration engine, interfacing with four elements: **environmental awareness and adaptation** (monitoring market shifts, resource constraints, competitor actions for *proactive* rather than reactive adjustment), **strategic task hierarchy** (from "Launch Product Campaign" down through workstreams to concrete tool integrations and API calls), **dynamic execution management** (an Execution Feedback loop feeding Plan Revision/Adaptation), and **integrated tool and knowledge orchestration** (external tools/APIs plus an Agent Memory/Knowledge Base).
  - **Three essential implementation strategies** (Ch 5, p. ~165):
    1. **Strategic task decomposition** — iterative prompting that identifies sequential *and parallel* subtasks, accounting for dependencies, resource requirements, time constraints, and risk factors. Worked example: "launch a new product marketing campaign" → market research, competitive analysis, content creation, distribution channel setup, performance monitoring, post-launch optimization; content creation → blog posts, infographics, video, social assets.
    2. **Dynamic execution and monitoring** — feedback mechanisms tracking progress against milestones, identifying bottlenecks or failures, triggering plan revisions; integration with external project management systems, API response monitoring, human feedback on task status.
    3. **Adaptive intelligence and learning** — if a critical dependency slips, the agent reschedules dependent tasks, reallocates resources, or proposes alternatives minimizing project impact, drawing on historical project data and learned patterns from the knowledge base.

- **Memory-Augmented agent** (Ch 5, p. ~166). Answers the limitation both prior architectures share: retaining context and learning beyond the current interaction. **Three memory types** (Ch 5, p. ~167):
  - **Working memory** — short-term, in-session, held *directly within the LLM prompt*: recent turns, current goals, task-specific data. Transient; cleared at session end or when the token limit is reached.
  - **Episodic memory** — a timestamped history of interactions and events. Recalls specific past experiences (a previous conversation, the outcome of a past action) for continuity and long-term personalization.
  - **Semantic memory** — structured factual information from documents, APIs, or databases: product specs, regulations, scientific facts. General knowledge and concepts, not events.
  - **Memory selection guide** — which store to query, by trigger, signal, and failure mode if skipped (Ch 5, p. ~167):

    | Type | Trigger condition | Query signal | Failure mode if skipped |
    |---|---|---|---|
    | Working | Active session; current turn in progress | LLM context window populated directly | Loss of immediate conversational coherence; agent forgets earlier turn context |
    | Episodic | Cross-session continuity needed; user references prior interaction | Vector similarity search over interaction history | Repeated history; agent treats each session as new, breaking personalization |
    | Semantic | Factual/domain knowledge required; not answerable from session alone | Keyword or semantic search over knowledge base | Hallucinated or stale answers; agent cannot ground claims in authoritative data |

  - **Memory-Augmented agent architecture** — eight components (Figure 5.3, Ch 5, p. ~167–169): (1) External Inputs, (2) Agent Core (cognition & reasoning, orchestrates memory interactions), (3) Working Memory as current prompt context, (4) **Vector Databases & Retrieval** bridging to long-term memory, ranking by semantic similarity, recency, or user preference, (5) Episodic Memory, (6) Semantic Memory, (7) External Databases/APIs/Documents feeding semantic memory via knowledge ingestion, (8) Agent Outputs closing the loop.
  - **Vector store choices named by the book** (Ch 5, p. ~168–169): **Chroma** (local development and prototyping), **Pinecone** (managed cloud, production-scale retrieval), **Weaviate** (open source, hybrid keyword + vector search).
  - **`MemoryAugmentedAgent` reference implementation** (Ch 5, p. ~169): `process_interaction()` = (1) `episodic_memory.search(query, user_id, limit=3)` → (2) inject retrieved memories into the prompt → (3) store `{user_id, query, response, timestamp}` back into episodic memory. `limit=3` is the book's value.

- **Planning versus decision-making** (Table 5.1, Ch 5, p. ~166) — the chapter's own comparison:

  | Capability | Autonomous Decision-Maker | Planning Agent |
  |---|---|---|
  | Focus | Immediate, tactical responses | Strategic, long-term goal achievement |
  | Scope | Single decisions at a time | Multi-step, coordinated processes |
  | Adaptation | Heuristics and conditional logic | Dynamic revision and learning loops |
  | Integration | Direct API calls for specific actions | Orchestrated tool workflows |
  | Memory use | Contextual within current session | Persistent, cross-project learning |

- **Comparative analysis of all three architectures** (Table 5.2, Ch 5, p. ~171–173) — strengths, use cases, and the book's stated trade-offs:

  | Agent type | Strength | Ideal use case | Limitations |
  |---|---|---|---|
  | Autonomous Decision-Making | Real-time response and flexibility | Customer service, triage, alerting | Emphasis on speed raises risk of hallucination or unsafe action absent proper constraints; lacks depth for strategic tasks |
  | Planning | Structured execution and adaptability | Project orchestration, complex workflows | Slower — analysis and design precede execution; demands robust monitoring/recovery control logic, adding architectural complexity |
  | Memory-Augmented | Long-term coherence and personalization | CRM systems, task handoff, continuity | Complex memory management and retrieval optimization; sensitive to retrieval design; risk of information overload |

## Key Concepts

- **Continuous cognitive loop** (Figure 5.1, Ch 5, p. ~156): environmental inputs → perception → cognition → action → new environmental state → back into perception. Action is not a terminus; it modifies the environment and generates the next perception, which is what makes the behavior autonomous rather than reactive (Ch 5, p. ~154, ~156).
- **80% autonomy ceiling**: the book's one concrete automation figure — by combining structured prompts and conditional logic, such a support bot "can operate autonomously for up to 80% of user queries," freeing human agents for nuanced problems (Ch 5, p. ~162). This is a claim about customer-support query volume, not a general design threshold.
- **Checkpointing is for human oversight, not crash recovery**: `execute_with_checkpoints()` belongs to the *guided* autonomous resolution strategy, the middle path between full autonomy and immediate escalation (Ch 5, p. ~153).
- **Memory consolidation/summarization** (Ch 5, p. ~171): long episodic entries are periodically summarized into concise profiles — the book's example is `Patient A, Type 2 Diabetes, responded well to diet changes (2025-01-15), struggled with medication adherence (2025-03-20)` — keeping vector retrieval efficient and avoiding performance degradation as history grows.
- **Case study: autonomous customer service in production** (Ch 5, p. ~161). Scenario: a premium customer reports intermittent business internet for two days with a critical presentation tomorrow. Flow: enhanced perception → autonomous reasoning (high priority, business impact, autonomous resolution approved) → `autonomous_safety_check` on the planned actions → execution of a 7-step workflow (check service area status, identify intermittent issues, schedule priority technician, temporarily upgrade service tier, provide mobile hotspot backup, set proactive monitoring, schedule follow-up call) → `monitor_resolution_effectiveness` → `update_autonomous_strategies`.
- **Case study: personalized healthcare assistant** (Ch 5, p. ~169–171). Demonstrates all three memory types in one system: working memory holds the immediate dialogue and patient tone; episodic memory stores symptom changes, successful coping strategies, medication adherence patterns, and emotional states; semantic memory holds drug interactions, disease protocols, and public health guidelines, continuously refreshed from clinical databases, medical journals, and regulatory documents. Personalization comes from *combining* all three — recommending a relaxation technique episodic memory shows previously worked, while tailoring dietary advice to allergies recorded in semantic memory. The worked dialogue: "I've been feeling very fatigued lately" is stored; a later "That diet change you suggested helped a bit, but I'm still tired" retrieves it by semantic relevance and injects it into the prompt (Ch 5, p. ~170).
- **Engineering best practices** (Ch 5, p. ~173–174), four named practices:
  - **Start modular** — separate perception, memory, planning, and action into distinct modules so that changing an LLM fine-tuning strategy in cognition does not break tool integration in action.
  - **Log everything (observability)** — LLM agents exhibit emergent and sometimes unpredictable behavior; log every stage from perception and prompt formulation through tool calls to final actions. **LangSmith** is named for debugging, testing, evaluating, and monitoring LLM applications.
  - **Avoid prompt bloat** — finite context windows mean excessive or irrelevant context degrades performance, increases latency, and raises cost. Mitigations: optimize retrieval pipelines, re-rank retrieved documents (detailed in Ch 6), use summarized memory entries.
  - **Ensure robustness and reliability** — design explicit fallbacks and escalation paths for undecidable or unsafe conditions: default to a human operator, request clarification, or roll back the action. Robust tool-integration error handling ties to Ch 7; prompt injection defense and secure tool access tie to Ch 4 (Ch 5, p. ~174–175).

## Code Pattern

```python
# The autonomous cognitive loop, as the book structures it (Ch 5, p. ~155)
class AutonomousCustomerServiceAgent:
    def cognitive_loop(self, user_message, context):
        system_state    = self.get_current_system_state()
        perception_data = enhanced_perception(user_message, context, system_state)  # 1. perceive
        decision_context = autonomous_reasoning(perception_data)                    # 2. reason
        results = autonomous_action_execution(decision_context)                     # 3. act
        self.learn_from_interaction(perception_data, decision_context, results)     # 4. learn
        self.update_session_memory(context.get("user_id"), decision_context, results)  # 5. persist
        return results
```

Note: `autonomous_safety_check()` is presented as a *design consideration* applied around planned actions (and shown in the case study between planning and execution), not as a step inside `cognitive_loop` itself (Ch 5, p. ~159, ~161).

## Anti-patterns

- **Speed without constraints**: the book's own stated trade-off for Autonomous Decision-Making agents — emphasis on real-time responsiveness "may increase the risk of hallucinations or unsafe actions, especially in the absence of proper constraints" (Ch 5, p. ~172). Guardrails, input validation, output filtering, and prompt injection defenses are the countermeasure (Ch 5, p. ~160).
- **Planning agents without control logic**: they "demand robust control mechanisms for monitoring progress and recovering from errors"; deploying one without monitoring and error recovery leaves nothing to catch the divergence between plan assumptions and reality (Ch 5, p. ~172).
- **Prompt bloat**: stuffing the context window with excessive or irrelevant retrieved content degrades performance, increases latency, and raises cost (Ch 5, p. ~174).
- **Unbounded episodic memory**: without memory consolidation/summarization, vector retrieval degrades in performance as interaction history grows (Ch 5, p. ~171). Memory-Augmented agents are also "sensitive to retrieval design" and require careful design to avoid information overload (Ch 5, p. ~172–173).
- **No escalation path**: "Not every problem can or should be solved autonomously." Agents without defined escalation criteria cannot gracefully defer on ambiguous situations, ethical dilemmas, or tasks beyond their capability (Ch 5, p. ~161).
- **Monolithic agent code**: skipping the perception/memory/planning/action module boundary makes debugging and enhancement harder and couples unrelated changes (Ch 5, p. ~174).

## Key Takeaways

1. Three architectures, three cognitive functions: decision-making (rapid, reactive responses), planning (multi-step objectives), memory (continuity via working, episodic, semantic) — jointly realizing the Ch 1 cognitive loop (Ch 5, p. ~175).
2. The Autonomous Decision-Making agent is the cornerstone; planning and memory are layered on top of it, not alternatives to it (Ch 5, p. ~149).
3. Strategy selection is a scored decision, not a binary: full autonomous resolution, immediate escalation, or guided autonomous resolution with checkpoints, chosen by weighted composite score (Ch 5, p. ~152).
4. Escalation criteria must be explicit and testable — the book names five with thresholds: safety violations, confidence < 0.8, complexity > 0.7, user escalation preference, business impact > 0.6 (Ch 5, p. ~160).
5. Memory-Augmented agents need all three memory types; the failure mode of skipping each is specified (lost coherence, broken personalization, hallucinated/stale answers) (Ch 5, p. ~167).
6. Planning is decomposition plus adaptation. Static plan execution is not planning — the hallmark is continuous re-evaluation and revision as conditions change (Ch 5, p. ~165).
7. Symbolic planning (STRIPS/PDDL) buys optimality guarantees at the cost of pre-modeling; LLM planning buys adaptability at the cost of guarantees. Pick by how well-understood and stable the domain is (Ch 5, p. ~163).
8. These architectures compose. The chapter's endpoint is Figure 5.4's integrated architecture, with multi-agent combination deferred to Ch 7 (Ch 5, p. ~173).

## Connects To

- **Ch1** ×10: the densest back-reference in the book — the cognitive loop, perception, reasoning, action, learning, the design considerations, and the customer service case study are all explicitly carried forward and extended here (Ch 5, p. ~149, ~150, ~151, ~153, ~154, ~155, ~156, ~157, ~159, ~161)
- **Ch2**: LLM selection guidelines per agent type, fine-tuning, model scaling (Ch 5, p. ~174)
- **Ch3** ×2: prompts shape the agent's cognitive behavior (Ch 5, p. ~152); prompt engineering and chain-of-thought are prerequisites for LLM-powered planning — ToT/Self-Ask are treated in depth there, not here (Ch 5, p. ~163)
- **Ch4** ×2: testing and continuous improvement (Ch 5, p. ~174); security/privacy, prompt injection defenses, secure tool access (Ch 5, p. ~175)
- **Ch6**: re-ranking retrieved documents for efficient context management (Ch 5, p. ~174)
- **Ch6** `[pos]`: the chapter's closing hand-off — "we turn to Information Retrieval and Knowledge agents" (Ch 5, p. ~176)
- **Ch7** ×2: combining these architectures into multi-agent workflows (Ch 5, p. ~173); Tool-Using agent error handling for tool integration (Ch 5, p. ~174)

## Companion Code
Repo: `30-Agents-Every-AI-Engineer-Must-Build/chapter05/`
- Runs without API key: `ch05_foundational_architectures__RUN_NO_KEY_SIMULATION.ipynb` (MockLLM) — notebook family is named `foundational_architectures`, not `cognitive_architectures`
- Provider variants: OpenAI GPT-4o / Claude Sonnet 4 / Gemini Flash 2.5 / Ollama DeepSeek local
- Key modules: `mock_llm.py`, `resilience.py`, `color_logger.py`
- Context: `USECASE.md`, `LLM_COMPARISON.md`, `TROUBLESHOOTING.md`, `LOCAL_LLM_SETUP.md`, `AGENTS.md`
