# Glosario canónico — senior-ai-engineer-mentor

> Este es el nombre OFICIAL de cada término. Todo milestone, modo y respuesta del mentor usa estos nombres, sin sinónimos sueltos. Si dudás cómo nombrar algo, esta es la fuente de verdad.

**Convención**: los 32 concept slugs (de `concepts.md`) tienen entrada con su slug. Los sub-términos técnicos no son slugs trackeados — son vocabulario de soporte. La columna **Hito** usa la numeración 1-6. Los nombres de hito canónicos son: Hito 1 Fundamentos, Hito 2 RAG & MCP, Hito 3 APIs & Microservicios, Hito 4 Orquestación, Hito 5 Multi-Agent, Hito 6 Producción & Governance.

---

## Hito 1 — Fundamentos

## Chain-of-Thought (CoT)
**También visto como:** "pensá paso a paso", reasoning prompting
**Definición:** Pattern de prompting que pide al LLM verbalizar su razonamiento antes de la respuesta final. Mejora lógica/math; inútil en classification simple.
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

## Few-Shot
**También visto como:** in-context examples, k-shot
**Definición:** Mostrarle al modelo 3-5 ejemplos curados en el prompt para bajar varianza y fijar formato. Más ejemplos NO es siempre mejor (costo + sobreajuste).
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

## Function calling
**También visto como:** tool calling, tool use (cuidado: ver "Tool use" abajo)
**Definición:** Protocolo por el cual el LLM emite un mensaje estructurado pidiendo ejecutar una función con args tipados; TU runtime la ejecuta y devolvés el resultado. El modelo nunca toca tu código.
**Hito:** 1
**Slug relacionado:** `function-calling`

## JSON mode
**También visto como:** structured outputs, constrained decoding, JSON output
**Definición:** Forzar al LLM a generar JSON válido contra un schema vía constrained decoding (no prompting). Garantía a nivel de sampler, no "pedir JSON y rezar".
**Hito:** 1
**Slug relacionado:** `json-mode`

## Memory tiers
**También visto como:** tipos de memoria, jerarquía de memoria
**Definición:** Working (turno actual, en el prompt), episodic (eventos pasados con timestamp, en store estructurado), semantic (conocimiento atemporal estable, en vector store/KG). Cada tier con storage, refresh y latency budget distintos.
**Hito:** 1
**Slug relacionado:** `memory-tiers`

## Parallel tool calls
**También visto como:** multi-tool, parallel function calls
**Definición:** El modelo emite múltiples tool_use en un solo turno; los ejecutás en paralelo (asyncio.gather) y devolvés todos los resultados juntos. Ojo con side effects/idempotencia.
**Hito:** 1
**Slug relacionado:** `function-calling`

## Plan-and-Execute
**También visto como:** plan-then-act
**Definición:** Loop que planifica todo upfront y después ejecuta, en contraste con ReAct (decide paso a paso). Conviene cuando el flow se conoce de antemano.
**Hito:** 1
**Slug relacionado:** `react-loop`

## Prompt patterns
**También visto como:** prompting techniques
**Definición:** Familia de patterns para estructurar prompts: PTCF, CoT, ToT, Few-Shot, Self-Consistency. Cada uno resuelve un problema distinto — se eligen midiendo, no mezclando todo.
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

## PTCF
**También visto como:** Persona-Task-Context-Format
**Definición:** Estructura de prompt de 4 bloques (Persona, Task, Context, Format). Sube consistencia en production agents con rol claro.
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

## ReAct loop
**También visto como:** loop cognitivo, agent loop, Thought-Action-Observation, ReAct puro
**Definición:** Loop Thought → Action → Observation que itera hasta Final Answer. Es una state machine implícita cuyo history crece linealmente (costo/latencia O(n²) sobre pasos). NO es "el algoritmo de los agentes", es una familia de loops.
**Hito:** 1
**Slug relacionado:** `react-loop`

## Reflexion
**También visto como:** self-critique loop
**Definición:** Variante de loop que agrega auto-crítica entre iteraciones. Mejora calidad a costo de latencia; no para real-time.
**Hito:** 1
**Slug relacionado:** `react-loop`

## Self-Consistency
**También visto como:** sampling + voting
**Definición:** Samplear N respuestas y votar la más frecuente. Sube robustez a costo ×N llamadas; típico en evals, no en producción.
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

## Structured Outputs (OpenAI)
**También visto como:** schema-constrained output
**Definición:** Feature de OpenAI que compila un schema a grammar/state machine y garantiza JSON válido + estructura. Agrega ~10% latencia (compilación cacheada).
**Hito:** 1
**Slug relacionado:** `json-mode`

## Tool use (Anthropic)
**También visto como:** tool_use blocks, function calling de Anthropic
**Definición:** Implementación de Anthropic del protocolo de tools, con tool_use blocks y tool_result blocks. `tool_choice` permite forzar una tool. Es function calling con matices propios.
**Hito:** 1
**Slug relacionado:** `function-calling`

## Tree-of-Thoughts (ToT)
**También visto como:** branch exploration
**Definición:** Pattern que explora múltiples ramas de razonamiento. Caro (×N caminos); solo vale cuando el costo de error es alto y no hay restricción de latencia.
**Hito:** 1
**Slug relacionado:** `prompt-patterns`

---

## Hito 2 — RAG & MCP

## ANN (Approximate Nearest Neighbor)
**También visto como:** búsqueda aproximada, HNSW/IVF/ScaNN
**Definición:** Algoritmos de índice (HNSW, IVF, ScaNN) que cambian exactitud por velocidad para escalar a millones de vectores. Es APROXIMADO — el doc correcto puede no caer en top-K si los params del índice están mal.
**Hito:** 2
**Slug relacionado:** `vector-search`

## Bi-encoder
**También visto como:** dual encoder, embedding model (en contexto de retrieval)
**Definición:** Modelo que embede query y doc por separado y compara con cosine. Permite indexar offline → se usa para retrieval inicial sobre millones.
**Hito:** 2
**Slug relacionado:** `re-ranking`

## BM25
**También visto como:** keyword search, sparse retrieval, búsqueda léxica
**Definición:** Algoritmo de ranking léxico (sparse) sobre términos exactos. Captura identidad exacta (IDs, códigos, jerga) que los embeddings "suavizan". Sigue siendo útil con LLMs.
**Hito:** 2
**Slug relacionado:** `hybrid-retrieval`

## Chunk
**También visto como:** fragmento, nodo (LlamaIndex), pasaje
**Definición:** Pedazo de documento que se embede e indexa por separado. Su tamaño y boundaries determinan qué tan bien recuperás contexto relevante.
**Hito:** 2
**Slug relacionado:** `chunking-strategy`

## Chunking strategy
**También visto como:** estrategia de chunking, splitting
**Definición:** Cómo partís un documento en chunks: fixed-size, sentence-window, recursive, semantic, hierarchical, late chunking. Se elige por estructura del doc Y patrón de query, NUNCA por chunk_size default.
**Hito:** 2
**Slug relacionado:** `chunking-strategy`

## Cosine similarity
**También visto como:** similitud coseno
**Definición:** Métrica de similitud que mide solo el ángulo entre vectores (normaliza magnitud). Si los embeddings están L2-normalizados, equivale a dot product (que es más rápido).
**Hito:** 2
**Slug relacionado:** `vector-search`

## Cross-encoder
**También visto como:** re-ranker model
**Definición:** Modelo que procesa (query, doc) JUNTOS en un forward pass, captando interacción semántica fina. No se puede pre-indexar → se usa solo para re-rank sobre top-K.
**Hito:** 2
**Slug relacionado:** `re-ranking`

## Dense retrieval
**También visto como:** semantic search, vector search, búsqueda semántica
**Definición:** Recuperación por similitud sobre embeddings densos. Capta paráfrasis y significado; falla en exact-match, jerga e IDs.
**Hito:** 2
**Slug relacionado:** `vector-search`

## Embeddings
**También visto como:** vectores densos, representaciones vectoriales
**Definición:** Vectores densos (768-3072 dims) que capturan significado: textos similares → vectores cercanos. Es la fundación de TODO RAG; cambiar de modelo post-deploy = re-indexar todo.
**Hito:** 2
**Slug relacionado:** `embeddings`

## Hybrid retrieval
**También visto como:** hybrid search, BM25+semántico, dense+sparse, retrieval híbrido
**Definición:** Combinar keyword search (BM25/sparse) con semantic search (dense) y fusionar con RRF o weighted sum. Default en RAG serio post-2024 — NO "nice-to-have".
**Hito:** 2
**Slug relacionado:** `hybrid-retrieval`

## MCP (Model Context Protocol)
**También visto como:** "USB-C de los LLMs", protocolo MCP
**Definición:** Estándar abierto (Anthropic, late-2024) para conectar LLMs a tools, resources y prompts vía servidores MCP. JSON-RPC 2.0 sobre stdio/SSE. NO es function calling — es discovery + ejecución entre apps y servidores.
**Hito:** 2
**Slug relacionado:** `mcp-protocol`

## MCP prompts
**También visto como:** prompts MCP
**Definición:** Templates parametrizables que un servidor MCP expone al usuario (UX guidance). Una de las tres primitivas MCP: tools (acción), resources (contexto), prompts (UX).
**Hito:** 2
**Slug relacionado:** `mcp-protocol`

## MCP resources
**También visto como:** resources MCP
**Definición:** Data leíble identificada por URI (`file://`, `db://`) que un servidor MCP expone; el cliente decide cuándo leerla. Es contexto, no acción.
**Hito:** 2
**Slug relacionado:** `mcp-protocol`

## MCP server
**También visto como:** servidor MCP
**Definición:** Proceso que expone tools/resources/prompts vía MCP. Local (stdio) o remoto (SSE/HTTP). Corre con los privilegios del proceso que lo lanzó — la seguridad la implementás vos (allowlist, consent, audit).
**Hito:** 2
**Slug relacionado:** `mcp-protocol`

## Re-ranking
**También visto como:** reordenamiento, second-stage ranking
**Definición:** Segunda pasada con un cross-encoder sobre el top-K del retrieval inicial para mejorar precisión. Retrieval da recall, re-ranking da precisión. ~50-150ms para top-20.
**Hito:** 2
**Slug relacionado:** `re-ranking`

## RRF (Reciprocal Rank Fusion)
**También visto como:** fusión por rankings
**Definición:** Algoritmo de fusión para hybrid retrieval que ignora scores y usa solo rankings: `score = Σ 1/(k + rank_i)`. Robusto, no necesita normalizar. Alternativa: weighted sum (más tunable, más frágil).
**Hito:** 2
**Slug relacionado:** `hybrid-retrieval`

## Sparse retrieval
**También visto como:** lexical retrieval, keyword retrieval
**Definición:** Recuperación basada en términos exactos (BM25). Opuesta a dense. Captura identidad exacta, no sinónimos.
**Hito:** 2
**Slug relacionado:** `hybrid-retrieval`

## Vector DB
**También visto como:** vector store, base de datos vectorial
**Definición:** DB para indexar y buscar embeddings (pgvector, Qdrant, Weaviate, Pinecone, Milvus). Se elige por 4 ejes: persistencia, filtros metadata, escalabilidad, operatoria. FAISS es librería, NO DB.
**Hito:** 2
**Slug relacionado:** `vector-search`

## Vector search
**También visto como:** dense retrieval, semantic search (ver "Dense retrieval")
**Definición:** Búsqueda por similitud (cosine/dot) sobre índices ANN. El concepto canónico de retrieval denso en la skill.
**Hito:** 2
**Slug relacionado:** `vector-search`

---

## Hito 3 — APIs & Microservicios

## Async patterns
**También visto como:** async/await, concurrencia, asyncio
**Definición:** Patrones de concurrencia (`async/await`) para hacer N llamadas LLM en paralelo sin bloquear. Async NO es "más rápido" — es mejor uso del I/O wait. Requiere Semaphore, gather con return_exceptions, timeout.
**Hito:** 3
**Slug relacionado:** `async-patterns`

## Backpressure
**También visto como:** control de flujo
**Definición:** Mecanismo para frenar la producción cuando el consumidor no da abasto, evitando saturar memoria/rate limits.
**Hito:** 3
**Slug relacionado:** `async-patterns`

## Batch API
**También visto como:** API de lotes
**Definición:** API async (≤24h SLA) con 50% de descuento (Anthropic, OpenAI). Para todo lo NO-realtime: embeddings de corpus, evals, summarización offline, datasets sintéticos.
**Hito:** 3
**Slug relacionado:** `cost-optimization`

## Cost optimization
**También visto como:** optimización de costos
**Definición:** Conjunto de prácticas para bajar costo LLM manteniendo calidad: model routing, batch API, prompt caching, response caching, streaming cancel, token budgets, eval-driven model selection. Es disciplina ongoing, no proyecto.
**Hito:** 3
**Slug relacionado:** `cost-optimization`

## Exponential backoff
**También visto como:** backoff con jitter, retry backoff
**Definición:** Estrategia de reintento donde el delay crece exponencialmente (+ jitter) tras cada 429. Respetá `Retry-After`. Nunca infinite retry.
**Hito:** 3
**Slug relacionado:** `rate-limits`

## Model routing
**También visto como:** model router, routing por costo/calidad
**Definición:** Un clasificador chico (o prompt simple a modelo barato) decide qué modelo invocar según la dificultad de la task. Modelo chico para fácil, grande para difícil. 60-80% saving típico.
**Hito:** 3
**Slug relacionado:** `cost-optimization`

## Prompt caching
**También visto como:** caching de prompt, cache de prefix
**Definición:** Cachear el prefix ESTABLE del prompt en el lado del PROVIDER para pagar solo tokens nuevos (Anthropic 90% off, OpenAI 50%). NO es Redis client-side. El prefix debe ser idéntico byte-a-byte; cualquier variable interpolada lo invalida.
**Hito:** 3
**Slug relacionado:** `prompt-caching`

## Rate limits
**También visto como:** límites de tasa, 429
**Definición:** Límites del provider sobre RPM (requests/min), TPM (tokens/min), a veces RPD/concurrent. Son constraint de arquitectura, no excepción. Se manejan con rate limiter client-side, backoff, fallback multi-provider.
**Hito:** 3
**Slug relacionado:** `rate-limits`

## Response caching
**También visto como:** cache de respuestas
**Definición:** Cache client-side (Redis) de respuestas a queries idénticas/similares (hash de query + versión del system prompt). 100% saving en hits; solo para queries idempotentes.
**Hito:** 3
**Slug relacionado:** `cost-optimization`

## RPM / TPM
**También visto como:** requests per minute / tokens per minute
**Definición:** RPM = cuántas llamadas hacés por minuto. TPM = cuántos tokens (input+output) procesás por minuto. El que llegue primero a su límite te tira 429.
**Hito:** 3
**Slug relacionado:** `rate-limits`

## Semaphore
**También visto como:** semáforo, concurrency limiter
**Definición:** `asyncio.Semaphore(N)` que limita llamadas simultáneas. N se calcula según RPM permitido y latencia media. OBLIGATORIO en producción — gather sin límite mata el rate limit.
**Hito:** 3
**Slug relacionado:** `async-patterns`

## SSE (Server-Sent Events)
**También visto como:** streaming, SSE streaming
**Definición:** Protocolo HTTP de streaming unidireccional server→client. Estándar de facto para streamear tokens LLM. Atraviesa proxies mejor que WebSocket; ojo con el buffering de Nginx (`X-Accel-Buffering: no`).
**Hito:** 3
**Slug relacionado:** `sse-streaming`

## Streaming cancel
**También visto como:** cancel propagation
**Definición:** Cancelar la llamada al provider cuando el user cierra la pestaña. Si NO lo hacés, el provider sigue generando y vos seguís PAGANDO tokens fantasma (10-20% del bill en chat).
**Hito:** 3
**Slug relacionado:** `cost-optimization`

## TTFB (Time To First Byte)
**También visto como:** latencia al primer token
**Definición:** Tiempo hasta el primer token. Prompt caching lo reduce ~85%. Métrica clave de UX en streaming.
**Hito:** 3
**Slug relacionado:** `prompt-caching`

---

## Hito 4 — Orquestación

## Checkpointing
**También visto como:** persistencia de state, checkpointer
**Definición:** Persistir el state del agente por step para sobrevivir crashes, hacer HITL, time-travel debugging y conversaciones persistentes. Non-negotiable en producción. Backends: MemorySaver (dev), SqliteSaver, PostgresSaver, RedisSaver.
**Hito:** 4
**Slug relacionado:** `checkpointing`

## DAG (StateGraph)
**También visto como:** grafo de estados, LangGraph DAG, grafo de agente
**Definición:** Grafo de estado explícito de un agente: nodos = funciones que mutan state, edges = transiciones (condicionales o fijas). Compila a StateGraph ejecutable con checkpointing/HITL/time-travel. State machine explícita para agents.
**Hito:** 4
**Slug relacionado:** `langgraph-dags`

## HITL (Human-in-the-Loop)
**También visto como:** revisión humana, human oversight
**Definición:** Pausar el grafo en un nodo (`interrupt_before`), esperar input/approval humano, y resumir con `update_state` + invoke. Requiere checkpointing persistente. Obligatorio para acciones irreversibles.
**Hito:** 4
**Slug relacionado:** `checkpointing`

## LangChain
**También visto como:** LCEL, langchain-basics
**Definición:** Framework de composición de chains con DSL declarativo (LCEL). Valor real con ≥3 LLM calls, multi-vendor swap, retrievers built-in. Overkill para 1-2 calls simples.
**Hito:** 4
**Slug relacionado:** `langchain-basics`

## LangGraph
**También visto como:** langgraph-dags
**Definición:** Framework de orquestación serio (2025-2026) para agentes como grafos de estado. EL estándar de producción. Agrega valor con branching, parallel nodes, HITL, checkpointing, observabilidad.
**Hito:** 4
**Slug relacionado:** `langgraph-dags`

## LCEL (LangChain Expression Language)
**También visto como:** pipe composition, Runnables
**Definición:** DSL de LangChain que compone Runnables con `|`: `prompt | llm | parser`. streaming/async/batch "gratis" en cualquier composición + tracing en LangSmith.
**Hito:** 4
**Slug relacionado:** `langchain-basics`

## LlamaIndex
**También visto como:** llamaindex-vs-langchain
**Definición:** Librería retrieval-first: parsers de docs sofisticados (LlamaParse), query engines, indexing strategies. Complementaria a LangChain, no rival. Patrón común: LlamaIndex como retriever DENTRO de un agente LangGraph.
**Hito:** 4
**Slug relacionado:** `llamaindex-vs-langchain`

## Reducer
**También visto como:** merge function, state reducer
**Definición:** Función que define cómo se mergean updates concurrentes a un campo del state (ej `Annotated[list, add_messages]`). Sin reducer, default es last-write-wins → pierde datos en parallel branches.
**Hito:** 4
**Slug relacionado:** `state-management`

## State management
**También visto como:** manejo de estado, state schema
**Definición:** Cómo modelás/propagás/mutás el state entre nodos. State es el CONTRATO entre nodos: schema explícito (TypedDict/Pydantic), reducers para concurrencia, immutability conceptual, bounded growth. State ≠ memory ≠ context window.
**Hito:** 4
**Slug relacionado:** `state-management`

## thread_id
**También visto como:** ID de conversación/workflow
**Definición:** Identificador que el checkpointer usa para una conversación/workflow específico (típicamente `user_id` o `session_id`). Permite resume, multi-turn chat y HITL con la misma "hebra".
**Hito:** 4
**Slug relacionado:** `checkpointing`

---

## Hito 5 — Multi-Agent

## Blackboard pattern
**También visto como:** pizarra, shared state coordination
**Definición:** Patrón donde múltiples agentes leen/escriben en una estructura compartida ("pizarra") en vez de comunicarse directo. Útil cuando la coordinación no se expresa fácil como protocolo. Riesgo: race conditions.
**Hito:** 5
**Slug relacionado:** `horizontal-network`

## Conflict resolution
**También visto como:** resolución de conflictos
**Definición:** Cómo el sistema decide cuando dos agentes proponen acciones incompatibles: voting, debate+arbiter, confidence-weighted, priority rules, HITL escalation. Parte del DISEÑO, no afterthought. Siempre debe existir un fallback que SIEMPRE resuelve.
**Hito:** 5
**Slug relacionado:** `conflict-resolution`

## Handoff prompting
**También visto como:** delegation prompting
**Definición:** El arte de qué contexto pasar al worker delegado: task description destilada, context filtrado, expected output format, constraints, return reason. NO pasar la conversación entera (diluye el foco del worker).
**Hito:** 5
**Slug relacionado:** `task-delegation`

## Hierarchical pattern
**También visto como:** supervisores anidados, jerarquía multi-agente
**Definición:** Supervisores anidados (top-supervisor → sub-supervisors → workers). Para sistemas grandes con dominios DISJUNTOS. Rara vez >2-3 niveles; 4+ es señal de mal modelado del dominio.
**Hito:** 5
**Slug relacionado:** `hierarchical-pattern`

## Horizontal network
**También visto como:** peer-to-peer agents, red horizontal, swarm
**Definición:** Agentes peer-to-peer sin jefe, coordinando por message passing, blackboard o negociación. Para simulaciones, debates, research, resilience. En apps de negocio típicas casi siempre es mala elección (deadlocks, debug miserable).
**Hito:** 5
**Slug relacionado:** `horizontal-network`

## Supervisor pattern
**También visto como:** patrón supervisor, orchestrator-workers, router agent
**Definición:** Un agente "jefe" recibe el input, delega a workers especializados, recolecta resultado y decide siguiente paso o termina. Topología centralizada. El DEFAULT razonable para multi-agent. Requiere descripciones de workers mutuamente excluyentes + límite de delegations.
**Hito:** 5
**Slug relacionado:** `supervisor-pattern`

## Task delegation
**También visto como:** delegación, routing a workers
**Definición:** La mecánica de a QUIÉN delega el supervisor (routing prompt, classifier, embedding similarity, reglas) y CON QUÉ contexto. La segunda parte (handoff prompting) es la que casi nadie piensa bien.
**Hito:** 5
**Slug relacionado:** `task-delegation`

## Voting
**También visto como:** votación, majority vote
**Definición:** Mecanismo de conflict resolution donde N agentes votan y gana la mayoría. Solo "fair" si los agentes son genuinamente diversos; si están entrenados parecido, es ECO, no robustez.
**Hito:** 5
**Slug relacionado:** `conflict-resolution`

---

## Hito 6 — Producción & Governance

## AAIP (Agencia de Acceso a la Información Pública)
**También visto como:** autoridad de datos AR
**Definición:** Autoridad argentina de protección de datos. Emitió la Disposición 2/2023 (recomendaciones para IA). Define criterios sobre anonimización, pseudonimización y transferencia internacional.
**Hito:** 6
**Slug relacionado:** `compliance-argentina`

## Compliance Argentina
**También visto como:** compliance AR
**Definición:** Marco legal AR para sistemas AI: Ley 25.326, Disposición 2/2023 AAIP, Decreto 836/2024. Cubre base legal, transferencia internacional, decisiones automatizadas y uso de IA en administración pública.
**Hito:** 6
**Slug relacionado:** `compliance-argentina`

## Compliance global
**También visto como:** compliance internacional
**Definición:** Marcos globales para sistemas AI: EU AI Act, GDPR, NIST AI RMF, ISO/IEC 42001, sectoriales (HIPAA, PCI-DSS, SOC2). Se mapea el sistema a categorías de riesgo y jurisdicciones.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## Cost attribution
**También visto como:** atribución de costos, chargeback
**Definición:** Tracking de costo LLM por tenant/feature/user/request vía metadata tagging. Habilita chargeback B2B, budget enforcement, Pareto de optimización, alertas de spike. Infra fundamental en LLM products B2B.
**Hito:** 6
**Slug relacionado:** `cost-attribution`

## Decreto 836/2024
**También visto como:** —
**Definición:** Norma AR que regula uso de IA en procedimientos administrativos del Estado Nacional (audit trail, supervisión humana, evaluación de impacto). Relevante si el cliente es gov AR.
**Hito:** 6
**Slug relacionado:** `compliance-argentina`

## Disposición 2/2023 AAIP
**También visto como:** Disp 2/2023
**Definición:** Recomendaciones de la AAIP para IA: transparencia (informar que hay AI), explicabilidad, revisión humana en decisiones con efecto significativo, auditoría algorítmica documentada.
**Hito:** 6
**Slug relacionado:** `compliance-argentina`

## DPA (Data Processing Agreement)
**También visto como:** acuerdo de tratamiento de datos
**Definición:** Contrato con el provider (Anthropic, OpenAI Enterprise) que regula el tratamiento de datos personales. Ayuda a cumplir transferencia internacional bajo Ley 25.326 y GDPR.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## EU AI Act
**También visto como:** Reglamento 2024/1689, AI Act
**Definición:** Regulación EU sobre RIESGO del sistema AI (vigente progresivo 2025-2027). 4 niveles: prohibited, high-risk (Annex III: empleo, educación, justicia...), limited (chatbots → transparencia), minimal. Distinto de GDPR.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## Evals
**También visto como:** evaluaciones, eval pipeline
**Definición:** Técnicas para medir si el agente funciona: golden datasets, métricas task-specific, LLM-as-judge, regression suites en CI, online evals. Infra non-negotiable. Sin esto, mejorás a ciegas.
**Hito:** 6
**Slug relacionado:** `evals`

## Faithfulness
**También visto como:** fidelidad (RAGAS)
**Definición:** Métrica de RAG (RAGAS) que mide que la respuesta NO alucina — que se sostiene en los docs recuperados. Junto con answer_relevancy y context precision/recall.
**Hito:** 6
**Slug relacionado:** `evals`

## GDPR
**También visto como:** Reglamento General de Protección de Datos
**Definición:** Regulación EU sobre datos personales (vigente 2018). Aplica EXTRATERRITORIALMENTE a quien procese datos de residentes EU. Art 22 = decisiones automatizadas; multas hasta 4% facturación global. Distinto de EU AI Act.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## Golden dataset
**También visto como:** dataset dorado, dataset de evaluación
**Definición:** Conjunto curado a mano de queries + outputs esperados (100-300 ejemplos) cubriendo casos típicos, edge cases y casos previamente fallados. Versionado en git, mantenido vivo con fallos de producción.
**Hito:** 6
**Slug relacionado:** `evals`

## ISO/IEC 42001
**También visto como:** ISO 42001, AI Management System
**Definición:** Estándar certificable de gestión de AI (2023). Equivalente a SOC2 pero para AI. Diferenciador en enterprise sales US/EU.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## Langfuse
**También visto como:** —
**Definición:** Herramienta de observability LLM open-source (self-host o cloud): tracing automático, costo per trace, evals integrados, dataset management. Multi-vendor. Alternativa: LangSmith.
**Hito:** 6
**Slug relacionado:** `observability`

## LangSmith
**También visto como:** —
**Definición:** Observability de LangChain Inc, integración nativa con LangChain/LangGraph, UI pulida pero LangChain-centric y de pago. Alternativa: Langfuse.
**Hito:** 6
**Slug relacionado:** `observability`

## Ley 25.326
**También visto como:** Ley de Protección de Datos Personales (AR)
**Definición:** Ley argentina de protección de datos. Aplica al RESPONSABLE del tratamiento sin importar dónde se procese: si tu empresa AR manda datos a OpenAI US, seguís siendo responsable. Requiere base legal (art 5) + base de transferencia internacional (art 12).
**Hito:** 6
**Slug relacionado:** `compliance-argentina`

## LLM-as-judge
**También visto como:** model-graded eval
**Definición:** Usar un LLM (DISTINTO al generador) para evaluar outputs en tareas cualitativas. Debe validarse contra labels humanos antes de confiar, con criterios numéricos en el prompt del judge. Si es el mismo modelo, sesgo positivo brutal.
**Hito:** 6
**Slug relacionado:** `evals`

## NIST AI RMF
**También visto como:** AI Risk Management Framework
**Definición:** Framework de governance US (voluntario, referencia). 4 funciones: Govern, Map, Measure, Manage. Útil como governance interno aunque no aplique regulatoriamente.
**Hito:** 6
**Slug relacionado:** `compliance-global`

## Observability
**También visto como:** observabilidad, tracing, tracing de agentes
**Definición:** Tracing de cada step del agente (LLM call, tool invocation, decisión) con timing/cost/IO/errores + metrics (latencia P50/P95/P99, costo/request, error rate, cache hit rate) + logs estructurados. Distinta del backend tradicional por tokens/costo/contexto.
**Hito:** 6
**Slug relacionado:** `observability`

## OWASP LLM Top 10
**También visto como:** OWASP GenAI Top 10
**Definición:** Lista de los 10 riesgos de seguridad LLM (2025). Prompt Injection es LLM01 (#1). Referencia obligatoria para threat modeling de agentes.
**Hito:** 6
**Slug relacionado:** `safety-prompt-injection`

## P50 / P95 / P99
**También visto como:** percentiles de latencia
**Definición:** Percentiles de latencia (mediana, cola alta). Métricas estándar de observability para SLA. P99 captura los peores casos que afectan UX.
**Hito:** 6
**Slug relacionado:** `observability`

## Prompt injection
**También visto como:** injection, jailbreak (subtipo)
**Definición:** Atacante manipula el prompt para alterar el comportamiento del LLM. Directa (user lo escribe) e indirecta (escondida en docs/web que el RAG procesa). El system prompt NO es boundary técnica → requiere defensa en capas.
**Hito:** 6
**Slug relacionado:** `safety-prompt-injection`

## Safety / prompt injection
**También visto como:** seguridad de agentes
**Definición:** Concepto canónico de defensas: input/output filtering, tool sandboxing, least privilege, structured outputs, separación de roles, dual LLM, HITL en críticos, monitoring. NO existe defensa perfecta; se baja el riesgo en capas.
**Hito:** 6
**Slug relacionado:** `safety-prompt-injection`

## Span
**También visto como:** —
**Definición:** Unidad de trabajo individual dentro de un trace (ej: una LLM call, una tool invocation). Un trace se compone de spans anidados.
**Hito:** 6
**Slug relacionado:** `observability`

## Trace
**También visto como:** traza
**Definición:** Registro completo de una ejecución del agente (todos los steps/spans con timing, cost, IO). La unidad sobre la que se debuggea y se atribuye costo en Langfuse/LangSmith.
**Hito:** 6
**Slug relacionado:** `observability`

## Dual LLM pattern
**También visto como:** untrusted/privileged split
**Definición:** Defensa de prompt injection: un LLM "untrusted" procesa el input del user/doc, otro "privileged" ejecuta acciones; el untrusted NUNCA accede a tools sensibles. Fuerte aislamiento a costo de latencia + costo 2x.
**Hito:** 6
**Slug relacionado:** `safety-prompt-injection`

## Tool sandboxing
**También visto como:** least privilege en tools
**Definición:** Dar a cada tool permisos mínimos (DB user read-only, paths allowlisted) para limitar el daño si hay injection. Capa clave de defensa.
**Hito:** 6
**Slug relacionado:** `safety-prompt-injection`
