# Catálogo canónico de conceptos — 36 conceptos en 7 hitos

Este es el **catálogo finito** que la skill trackea. Cada slug es el `topic_key` usado en engram:
`skill/ai-engineer-mentor/mastery/{slug}`.

Columna **Capítulo libro** referencia notebooks en `C:/Users/Dell/Agus/Ai Agents Imran Ahmad/30-Agents-Every-AI-Engineer-Must-Build`.
Columna **Gap externo** sólo se completa si el libro NO cubre bien el concepto.

---

## Hito 0 — Integración práctica: webhook + agente en producción real (4)

Hito práctico, sin capítulo en el libro (100% gap externo) — pensado para acompañar un TP/proyecto real de
integración con una plataforma externa (mensajería, CRM, lo que sea), no solo el diseño del agente en sí.

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `webhook-vs-polling` | 0 | Cómo enterarte de eventos externos: push (webhook) vs pull (polling) — idempotencia y reintentos | — | Doc de reintentos del proveedor específico que estés integrando |
| `agent-conversation-memory` | 0 | Usar el historial de una plataforma externa como memoria del agente, curado para no imitar artefactos de formato | — | — |
| `tool-output-validation` | 0 | Validación server-side de argumentos de tool-calling antes de ejecutar efectos — el system prompt no es garantía | — | OWASP LLM Top 10 (Excessive Agency): https://genai.owasp.org/ |
| `external-platform-auth-patterns` | 0 | API key vs sesión CLI/OAuth vs MCP — cuál usar según si hay un humano presente o el proceso es desatendido | — | Doc de secrets management del runtime que uses (Supabase, AWS, Vercel, etc.) |

## Hito 1 — Fundamentos (5)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `react-loop` | 1 | El loop cognitivo Thought→Action→Observation: cómo un agente "piensa" iterando con herramientas | chapter01 + chapter05 | — |
| `json-mode` | 1 | Forzar al LLM a devolver JSON válido constreñido por schema, en vez de texto libre que después parseás con regex | chapter01 (menciona) | OpenAI Structured Outputs: https://platform.openai.com/docs/guides/structured-outputs |
| `function-calling` | 1 | El protocolo por el cual el LLM "pide" ejecutar una función con args tipados, y vos ejecutás y le devolvés el resultado | chapter01 + chapter07 | Anthropic tool use: https://docs.anthropic.com/en/docs/build-with-claude/tool-use |
| `memory-tiers` | 1 | Memoria working (turno actual) vs episodic (eventos pasados) vs semantic (conocimiento estable) y cuándo usar cada una | chapter05 | — |
| `prompt-patterns` | 1 | PTCF (Persona-Task-Context-Format), Chain-of-Thought, Tree-of-Thought, Few-Shot — cuál usar para qué problema | chapter03 | — |

## Hito 2 — Retrieval & MCP (6)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `chunking-strategy` | 2 | Cómo partir documentos: fixed-size vs semántico vs sentence-window vs hierarchical, y por qué importa más de lo que parece | chapter06 | — |
| `embeddings` | 2 | Vectores densos que capturan significado: qué modelos elegir, dimensionalidad, costo, refresh strategy | chapter06 | — |
| `vector-search` | 2 | Búsqueda por similitud (cosine, dot) sobre índices ANN (FAISS, HNSW): qué tradeoffs tiene vs exact search | chapter06 | — |
| `hybrid-retrieval` | 2 | Combinar BM25 (keyword) con semantic search para cubrir queries que pura vector no resuelve | — | LangChain Hybrid Search: https://python.langchain.com/docs/integrations/retrievers/ |
| `re-ranking` | 2 | Segunda pasada con un modelo más caro (cross-encoder) sobre el top-K del retrieval para mejorar precisión | — | Cohere Rerank: https://docs.cohere.com/docs/rerank-overview |
| `mcp-protocol` | 2 | Model Context Protocol: estándar abierto para conectar LLMs a tools/data sources, qué resuelve vs function calling ad-hoc | chapter01 (MCPRegistry mención) | Anthropic MCP spec: https://modelcontextprotocol.io/specification |

## Hito 3 — Async & Costos (5)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `async-patterns` | 3 | async/await, concurrencia controlada, semáforos, backpressure — cuando el agente hace 50 llamadas en paralelo sin matar la API | — | Python asyncio docs + OpenAI async client |
| `sse-streaming` | 3 | Server-Sent Events para streaming de tokens: cuándo conviene vs respuesta completa, manejo de errores parciales | — | OpenAI streaming: https://platform.openai.com/docs/api-reference/streaming |
| `rate-limits` | 3 | TPM, RPM, retries con exponential backoff, distribución entre keys, fallback a otros providers | chapter04 (menciona) | Anthropic rate limits: https://docs.anthropic.com/en/api/rate-limits |
| `prompt-caching` | 3 | Cachear el prefix estable del prompt para bajar costo 90% y latencia: cuándo se invalida, cómo estructurar | — | Anthropic prompt caching: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching |
| `cost-optimization` | 3 | Routing por costo/calidad, batch API, modelo chico para router + modelo grande para casos difíciles | chapter04 | OpenAI Batch API: https://platform.openai.com/docs/guides/batch |

## Hito 4 — Frameworks (5)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `langchain-basics` | 4 | Chains, Runnables, LCEL — qué problema resuelve LangChain y cuándo es overkill | chapter02 | LangChain docs: https://python.langchain.com/docs/introduction/ |
| `langgraph-dags` | 4 | Grafos de estados explícitos para agentes: nodos, edges condicionales, por qué supera al loop ad-hoc | chapter02 (menciona) | LangGraph docs: https://langchain-ai.github.io/langgraph/ |
| `state-management` | 4 | Cómo modelar y propagar estado entre nodos de un grafo de agente sin explotar el contexto | chapter07 | LangGraph state schemas |
| `checkpointing` | 4 | Persistir estado del agente entre turnos: human-in-the-loop, resume después de crash, time-travel debugging | — | LangGraph checkpointers: https://langchain-ai.github.io/langgraph/concepts/persistence/ |
| `llamaindex-vs-langchain` | 4 | Cuándo conviene LlamaIndex (retrieval-first) vs LangChain (orquestación general) vs LangGraph (control de flujo) | chapter02 | LlamaIndex docs: https://docs.llamaindex.ai/ |

## Hito 5 — Multi-Agente (5)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `supervisor-pattern` | 5 | Un agente "jefe" que rutea a workers especializados según el tipo de tarea | chapter07 + chapter15 | LangGraph supervisor: https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/ |
| `hierarchical-pattern` | 5 | Supervisores anidados (jerarquía de N niveles) para tareas complejas con sub-equipos | chapter15 | — |
| `horizontal-network` | 5 | Agentes peer-to-peer sin jefe, coordinando por message passing o blackboard | chapter17 | — |
| `task-delegation` | 5 | Cómo el supervisor decide a quién delegar: routing prompt vs clasificador entrenado vs reglas | chapter07 | — |
| `conflict-resolution` | 5 | Cuando dos agentes proponen acciones incompatibles: voting, debate, arbiter, HITL escalation | chapter07 | — |

## Hito 6 — Producción / Eval / Compliance (6)

| Concepto (slug) | Hito | Descripción 1-línea | Capítulo libro | Gap externo |
|---|---|---|---|---|
| `evals` | 6 | Cómo medir si tu agente funciona: golden datasets, LLM-as-judge, métricas task-specific, regression suites | — | OpenAI Evals: https://github.com/openai/evals + Anthropic eval cookbook |
| `observability` | 6 | Tracing de cada step del agente, costo por request, latencia P95/P99, debugging de multi-agent traces | chapter04 (menciona) | Langfuse: https://langfuse.com/docs ; LangSmith: https://docs.smith.langchain.com/ |
| `safety-prompt-injection` | 6 | Defensas contra prompt injection directo e indirecto, sandboxing de tools, principle of least privilege | chapter04 + chapter10 | OWASP LLM Top 10: https://genai.owasp.org/ |
| `compliance-argentina` | 6 | Ley 25.326 Protección Datos Personales, residencia de datos, transferencia internacional, contexto AR/LATAM | — | Argentina AAIP: https://www.argentina.gob.ar/aaip |
| `compliance-global` | 6 | GDPR (EU), EU AI Act (high-risk systems), HIPAA (US health), PCI-DSS, SOC2 — qué exige cada uno a un agente LLM | chapter04 + chapter09 + chapter12 | EU AI Act: https://artificialintelligenceact.eu/ |
| `cost-attribution` | 6 | Tracking de costo por tenant/feature/user, budget enforcement, alertas, chargeback en B2B | chapter04 (menciona) | Langfuse cost tracking docs |

---

## Total: 36 conceptos

Distribución: Hito 0 (4) + Hito 1 (5) + Hito 2 (6) + Hito 3 (5) + Hito 4 (5) + Hito 5 (5) + Hito 6 (6) = **36**.

Cualquier concepto fuera de esta lista no es trackeado por la skill (por diseño — granularidad fija evita scope creep). Si querés agregar uno, editá este archivo Y `SKILL.md` (mapa de hitos).
