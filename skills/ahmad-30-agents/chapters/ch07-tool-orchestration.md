# Chapter 7: Tool Manipulation and Orchestration Agents

## Core Idea
Tool use is not just calling functions — it's a full architecture: a reasoning core that plans which tools to call, a registry that defines available tools with typed schemas, an execution engine that handles failures, and a guarded tool chest that enforces safety. The chapter is organized as three progressive patterns: Tool-Using agents (a single agent invoking external functions), chain-of-agents orchestrators (multiple specialists coordinated by a manager), and agentic workflow systems (long-running stateful processes with human oversight) (Ch 7, p. ~204).

## Frameworks Introduced

- **Tool-Using Agent Architecture** — a Think, Plan, Act cycle over 4 components (Figure 7.1, Ch 7, p. ~206):
  1. **Reasoning core**: the "brain". Parses intent and formulates a step-by-step plan (Think and Plan). Consults the registry to plan validly.
  2. **Tool registry**: catalog of available tools with name, description, and input/output schemas. Those schemas are explicit contracts telling the agent exactly what to supply and what to expect (Ch 7, p. ~206).
  3. **Execution engine**: the "foreman" for the Act stage — manages state between steps, handles retries, propagates errors. Decoupled from the reasoning core (Ch 7, p. ~206).
  4. **Tool chest**: the actual functions, each with a single clearly defined responsibility and wrapped in its own safety logic (timeouts, exception handling, input validation). The running example is a four-function chest: `load_csv`, `aggregate`, `plot_chart`, `validate` (Ch 7, p. ~207).

- **Function-Calling Architecture Patterns** (five recurring patterns, Ch 7, p. ~208):
  - **Interface schemas as contracts**: each tool registers a precise input–output signature via JSON schema or typed models such as Pydantic. In the book's implementation, Python type hints (`file_path: str`, `df: pd.DataFrame`) *are* the schema contract (Ch 7, p. ~209).
  - **Separation of decision and execution**: reasoning decides what and when; the execution layer invokes. Improves traceability and lets both layers evolve independently.
  - **Reactive vs. deliberative invocation**: reactive maps a request directly to one tool; deliberative formulates a plan (load → filter → aggregate → visualize) and executes tools at each step.
  - **Safe wrappers and fallback logic**: retries, timeouts, exception handling per invocation, plus graceful degradation (substitute a bar chart when the pie chart function fails).
  - **Dynamic discovery via metadata**: tools publish descriptions, I/O types, and operational status so agents can discover and rank them at runtime without manual reconfiguration.

- **Tool selection funnel** — three progressive stages that turn a large registry into one verified executable choice (Figure 7.2, Ch 7, p. ~211): **intent classification** (coarse filtering of the tool category), **semantic search** (embed request and tool metadata, rank by semantic distance, discard below a confidence threshold such as 0.7), **constraint filtering** (deterministic checks on input types, user permissions, historical failure status). For multi-step tasks the funnel is invoked per plan step, and dynamic reranking reinitiates it when a tool fails. Five implementation strategies: intent classification/template matching, embedding-based similarity search, constraint-based filtering (Ch 7, p. ~212), plan-driven tool assignment, and dynamic reranking with feedback (Ch 7, p. ~213).

- **Error handling as a first-class concern** — four failure modes: input validation errors, runtime failures, **semantic mismatches** (the tool succeeds but the result misses the intent — e.g. "top campaigns by spend" charted alphabetically), and tool unavailability (Ch 7, p. ~215). Six layered recovery strategies: safe invocation wrappers (with targeted retry such as exponential backoff for network errors), fallback tool chains, confidence-based switching, **failure memory** (a circuit breaker that temporarily marks a repeatedly failing tool unavailable), escalation paths that hand a human the full context, and comprehensive logging and telemetry (Ch 7, p. ~215–216).

- **Chain-of-Agents (CoA) Orchestration** — a manager agent coordinates a team of specialists, each with its own domain expertise and tool chest (Ch 7, p. ~217). The team operates under a **cooperation protocol** built on four architectural pillars (Ch 7, p. ~218):
  - **Clearly defined roles and capabilities** — e.g. `PlannerAgent`, `DataFetcherAgent`, `ValidatorAgent`, each granted only the tools its role needs.
  - **A common communication infrastructure** — RPC, message queues, or a shared memory store for structured messages.
  - **Shared context or memory** — the single authoritative workspace so every agent works from the same facts.
  - **Execution orchestration** — the manager delegates, manages dependencies, and monitors progress.
  - Concretely the protocol defines four elements (Table 7.1, Ch 7, p. ~219): message format (a shared envelope with `sender`, `recipient`, `task_id`, `payload`), role declaration, task delegation scheme, and status signaling (`pending`, `running`, `done`, `error`). These map onto **MCP and A2A** from Chapter 6 — the cooperation protocol is the practical expression of what those standardize at the wire level (Ch 7, p. ~219).

- **Memory-augmented multi-agent architecture** (Figure 7.3, Ch 7, p. ~220): **working memory** is the active scratchpad the Agent Core curates for the current task; **long-term memory** is the shared persistent library, typically a vector database, split into **episodic memory** (logbook of events, agent conversations, tool outputs) and **semantic memory** (durable factual knowledge — company descriptions, product catalogs, regulatory guidelines). On a new task the Agent Core retrieves from long-term into working memory so downstream agents share the planner's foundational facts.

- **Human-in-the-Loop Coordination**: agentic workflows with defined pause points where human approval gates execution. In the e-commerce workflow this is an explicit "Step 2.5" escalation gate — when the LLM risk analyst returns medium or high risk, an `input()` prompt halts the program and requires an explicit approve/reject from a human operator (Ch 7, p. ~228). Beyond linear chains, workflows need conditional branching, waiting states for external events, and rollback paths — the book names **LangGraph** and **CrewAI** as the orchestration engines designed for this (Ch 7, p. ~229).

## Key Concepts

- **Tool schema = contract**: the schema is not documentation — it's enforcement. Explicit, reliable contracts are what make precise planning possible (Ch 7, p. ~206).
- **Observability for tool calls**: "You cannot fix what you cannot see." Every action, error, and recovery attempt must be logged — the data identifies unreliable tools and systemic weaknesses, not just individual bugs (Ch 7, p. ~216).
- **Graceful degradation**: when a tool is unavailable, fall back to an alternative tool chain rather than aborting the task — e.g. `fetch_stock_data_from_api_A` fails, so the agent retries via `..._api_B` (Ch 7, p. ~215).
- **State passing is what makes multi-step agentic processes coherent**: in `data_viz_agent`, a single `data_state` variable carries the DataFrame from tool to tool across the plan (Ch 7, p. ~217).
- **Single responsibility per tool**: modular functions with one clear job simplify building, testing, and extending — new plotting tools can be added without touching the agent's core logic (Ch 7, p. ~209).

## Code Pattern

The book's registry metadata is name, description, input/output schema, and operational status — the last one is what enables dynamic discovery and the failure-memory circuit breaker (Ch 7, p. ~207).

```python
# Tool registry pattern (metadata per the book: name, description, I/O schema, status)
tool_registry = {
    "load_csv":   {"schema": LoadCSVSchema, "timeout": 30, "status": "available"},
    "aggregate":  {"schema": AggSchema,     "timeout": 30, "status": "available"},
    "plot_chart": {"schema": PlotSchema,    "timeout": 30, "status": "degraded"},  # pruned by constraint filter
}

def call_tool(tool_name: str, args: dict):
    spec = tool_registry[tool_name]
    validated = spec["schema"](**args)  # raises ValidationError if invalid
    return execute_with_timeout(tool_name, validated, spec["timeout"])
```

The chapter's own worked example is the Think → Plan → Act loop of `data_viz_agent`: `parse_query` extracts a structured intent (metric, dimension, chart type), the intent drives construction of a plan as a list of tool calls, then the plan is executed step by step with `data_state` threaded through (Ch 7, p. ~216–217).

## Anti-patterns

- **Untyped tool arguments**: passing raw dicts without schema validation. The registry's I/O schemas exist precisely to make arguments and return values predictable (Ch 7, p. ~208).
- **No execution engine**: calling tools directly from reasoning code without retry, timeout, or error handling. "Failure is not a possibility; it is an inevitability" — designing an agent without error handling is "like building a skyscraper on a weak foundation" (Ch 7, p. ~214).
- **Retrying a tool that is clearly down**: without failure memory acting as a circuit breaker, the agent burns cycles re-selecting a tool that has already failed repeatedly (Ch 7, p. ~215).
- **Treating success as correctness**: semantic mismatches pass every runtime check yet produce useless output. Only confidence-based switching or human review catches them (Ch 7, p. ~215).
- **Treating human review as a failure path**: it is a planned, first-class branch in the control flow, not an exception handler (Ch 7, p. ~224).

## Key Takeaways

1. Tool-using agent requires 4 components: reasoning core + tool registry + execution engine + tool chest (Ch 7, p. ~206).
2. Tool schemas are contracts, not documentation — enforce them strictly (Ch 7, p. ~208).
3. Selection scales through a funnel, not a bigger prompt: intent classification → semantic search → constraint filtering (Ch 7, p. ~211).
4. Layered recovery beats prevention: safe wrappers, fallback chains, confidence switching, failure memory, escalation, telemetry (Ch 7, p. ~215).
5. Chain-of-Agents is the pattern for multi-domain orchestration, and it only works with an explicit cooperation protocol plus shared memory (Ch 7, p. ~218).
6. Disagreement between specialists is a feature; architectural strength is the capacity to resolve it productively and auditably (Ch 7, p. ~222).
7. Log every tool call, decision, confidence score, and rationale — it is both the debugging surface and the compliance audit trail (Ch 7, p. ~216, ~231).

## Case Study: Multi-Agent Insurance Claims Workflow (Claim CLM-4821)

A claim must travel from submission to resolution through checks and decision points, managed by a team of specialized agents coordinated inside a structured state machine. The architecture balances efficiency against risk: routine low-risk claims are automated for speed while complex or high-risk cases escalate to expert human review. Five agents divide the work. The **Intake agent** uses OCR and NLP to digitize submitted claim forms. The **Validator agent** gatekeeps — checking policy status, cross-referencing fraud databases, confirming documentation completeness. The **Classifier agent** assesses type, urgency, and risk level once the claim is validated. The **Payout agent** calculates settlement and processes payment for approved low-risk claims. The **Escalation agent** monitors for high-risk flags or low-confidence scores and routes to a human (Ch 7, p. ~229).

The state machine (Figure 7.5) runs Intake → Validating → Assessing Risk → Processing Payout → Closed: Approved on the happy path, with branches to Pending Human Review and Closed: Rejected (Ch 7, p. ~230). Transitions are driven by explicit guard conditions (Table 7.2, Ch 7, p. ~231): `claim_amount > threshold` routes to human approval; validation failure transitions to Closed: Rejected; `confidence_score < 0.85` escalates to Pending Human Review.

The CLM-4821 walkthrough makes this tangible (Ch 7, p. ~230–231). It is a water-damage home policy claim for $8,400 under policy POL-992317 (active, no fraud flag). Intake extracts `claim_id`, `policy_id`, `type = water_damage`, `amount = 8400.00`; the guard "all required fields populated" advances to Validating. The validator confirms the policy is active with no fraud flag. On the approved path the classifier returns `confidence_score = 0.91`, risk low — clearing the 0.85 guard — and the payout agent settles $8,400, closing as Approved. On the escalation path the classifier instead returns `confidence_score = 0.79` with a high-risk flag, tripping the guard and rerouting to Pending Human Review, where a reviewer either approves (back to Processing Payout) or rejects.

**What it exercises**: the agentic workflow system and HITL coordination frameworks — human review as a planned, first-class branch in the control flow rather than a failure path. **Lesson**: at each transition the system records agent reasoning, tools used, human decisions, and record updates, producing a complete audit trail essential for compliance, debugging, and postmortem analysis. Governance and auditability matter as much as throughput in regulated, long-running processes (Ch 7, p. ~231).

Note the book's own framing of implementation depth across its three case studies: the data visualization assistant is fully executable code; the market intelligence platform mixes runnable specialist agents with an architectural description of the memory and conflict-resolution layers; the insurance claims workflow is a state machine and governance architecture rather than running code (Ch 7, p. ~231).

### Related: Conflict resolution mechanisms

The chapter develops arbitration as its own framework (Figure 7.4, Ch 7, p. ~223), applied to the market intelligence system where `NewsAgent`, `FinancialAgent`, and `SentimentAgent` each return the same envelope shape `{"source", "status", "data"}` and write competing signals into shared memory; the manager writes each result into episodic memory, and an `LLMAnalystAgent` later combines fresh episodic findings with durable semantic background to produce the report (Ch 7, p. ~221–222).

Four arbitration stages (Ch 7, p. ~224): **conflict detection** (semantic similarity between competing outputs; below a threshold such as 0.7, a conflict is flagged), **automated arbitration** (an impartial arbiter agent consults a trusted knowledge base, re-examines source data, or applies a learned policy — sometimes synthesizing a new output combining both reports), **confidence-based consensus** (the arbiter emits a calibrated confidence score evaluated against a policy threshold such as 95%), and **human escalation** when confidence is low. Disagreement is a sign of robust diverse analysis, not a defect — architectural strength is measured by capacity to resolve conflict productively, with all inputs, intermediate decisions, confidence scores, and rationales logged (Ch 7, p. ~222, ~224).

The implementation is deliberately rudimentary: `ManagerAgent._synthesize_report` computes `conflict_score = abs(sentiment_score - (stock_change / 10))` and flags a conflict above 0.5. The book explains the normalization — `sentiment_score` is already bounded to [−1, 1], and dividing a percentage `stock_change` by 10 maps a ±10% daily swing onto the same scale, so the 0.5 threshold is a 5-point gap between sentiment and stock movement (Ch 7, p. ~225).

*Code discrepancy in the book*: `_synthesize_report` reads `financial_data['change_pct']` and `sentiment_data['sentiment_score']`, but the specialist agents as printed return `pe_ratio` / `revenue_growth` / `debt_to_equity` / `last_close` and `score` / `label` / `evidence` respectively (Ch 7, p. ~221–222, ~225). The keys do not line up — treat the snippet as illustrative, not runnable as printed.

## Connects To

- **Ch6** ×2 — declared: MCP and A2A formalize this chapter's cooperation protocol at the wire level; Ch6 covers structure and negotiation, Ch7 applies it (Ch 7, p. ~219). **⚠️ Attribution conflict inside the book**: Ch7 states MCP/A2A were "introduced in Chapter 6", but Ch2 credits Chapter 1 — and Ch1, p. ~47 is where they are actually defined. Ch2 is right
- **Ch8** — declared forward: data analysis and reasoning agents apply these orchestration patterns to dataset exploration, visualization recommendation, and statistical reasoning (Ch 7, p. ~232)
- **Ch9** `[pos]` — declared by position, no number: "Later chapters introduce software development agents" that write, test, and maintain code alongside human developers (Ch 7, p. ~232)
- **Ch5**: Planning agent provides the reasoning core (inferred — not stated in Ch7)
- **Ch4**: HITL coordination connects to deployment safety gates (inferred — not stated in Ch7)

## Companion Code
Repo: `30-Agents-Every-AI-Engineer-Must-Build/chapter07/`
- Runs without API key: `ch07_tool_orchestration__RUN_NO_KEY_SIMULATION.ipynb` (MockLLM)
- Provider variants: OpenAI GPT-4o / Claude Sonnet 4 / Gemini Flash 2.5 / Ollama DeepSeek local
- Key modules: `helpers/mock_llm.py`, `helpers/resilience.py`, `helpers/color_logger.py`
- Context: `USECASE.md`, `LLM_COMPARISON.md`, `TROUBLESHOOTING.md`
