# Diagnostic Probes — Chequeos rápidos por concepto

**Cuándo se carga**: lazy, cuando el mentor necesita medir mastery de un concepto sin hacer una entrevista completa (~30 seg, no minutos).

**Cómo usar**: una pregunta corta + signals de respuesta (good / partial / bad) + decisión post-respuesta sobre mastery level.

**Distinción vs `interview-questions-bank.md`**: estas son cortas (1-2 oraciones) para chequeo al vuelo. El interview bank son entrevistas completas.

**Voz**: Gentleman Rioplatense, voseo, directo. Sin warm-up — son diagnósticos.

---

## react-loop

### Probe 1
**Pregunta:** "En 1 oración, ¿qué hace un loop ReAct?"
**Good signals:** menciona iteración Thought→Action→Observation hasta resolver.
**Partial signals:** "razona y actúa" sin loop iterativo o sin observation.
**Bad signals:** confunde con function calling o dice "el agente piensa".
**Decisión:** good → mantén explored/practiced. Partial → degrade un nivel si estaba practiced+. Bad → forzar a unknown.

### Probe 2
**Pregunta:** "¿Qué tenés que poner para evitar que un agente ReAct loopee infinito?"
**Good signals:** max_iterations y/o timeout total, fallback claro.
**Partial signals:** "el LLM sabe cuándo parar" + alguna mención de límite.
**Bad signals:** "nada, eventualmente termina".
**Decisión:** good → señal de practiced. Bad → degradar.

### Probe 3
**Pregunta:** "Diferencia en 1 oración entre ReAct y function calling."
**Good signals:** ReAct = patrón orquestación iterativa, function calling = mecanismo de llamada.
**Bad signals:** "son lo mismo".
**Decisión:** ata mastery a esta distinción — bad → forzar unknown.

---

## json-mode

### Probe 1
**Pregunta:** "¿Diferencia entre JSON mode y Structured Outputs?"
**Good signals:** JSON mode = válido sintácticamente; Structured Outputs = schema-constrained en decode time.
**Partial:** "los dos devuelven JSON" sin distinción.
**Bad:** "no sé / son lo mismo".
**Decisión:** good → explored+. Bad → unknown.

### Probe 2
**Pregunta:** "Si pedís JSON solo en el prompt sin structured outputs, ¿qué puede salir mal?"
**Good signals:** menciona markdown wrappers, trailing commas, campos faltantes, parsing frágil.
**Bad:** "nada, GPT-4 ya devuelve JSON bien".
**Decisión:** bad → degradar.

### Probe 3
**Pregunta:** "¿Cuándo NO usarías JSON mode?"
**Good signals:** output narrativo, multi-format, schema dinámico.
**Bad:** "siempre conviene JSON mode".
**Decisión:** señal de senior si distingue.

---

## function-calling

### Probe 1
**Pregunta:** "Si exponés 30 funciones, ¿qué pasa?"
**Good signals:** routing degrada, context inflado, latencia/costo suben. Solución: subsets dinámicos o router.
**Bad:** "está bien, el LLM elige".
**Decisión:** bad → señal de no entender.

### Probe 2
**Pregunta:** "Tu LLM llama `delete_user(id=123)` por error. ¿Cómo lo evitás?"
**Good signals:** HITL gate, confirmation token, soft delete, RBAC.
**Partial:** "valido el id".
**Bad:** "le digo en el prompt que no borre".
**Decisión:** good → practiced. Bad → degradar.

### Probe 3
**Pregunta:** "Parallel function calling vs sequential — ¿cuándo cada uno?"
**Good signals:** parallel para fan-out independiente, sequential cuando hay dependencias.
**Bad:** desconoce parallel.
**Decisión:** ata.

---

## memory-tiers

### Probe 1
**Pregunta:** "Working, episodic, semantic — ¿cuál es cuál en un agente?"
**Good signals:** working=turno actual, episodic=eventos pasados, semantic=knowledge estable.
**Partial:** sabe 2 de 3.
**Bad:** confuso o "es todo lo mismo".
**Decisión:** ata.

### Probe 2
**Pregunta:** "¿Por qué no metés todo en el contexto y listo?"
**Good signals:** context window limit, costo lineal, needle in haystack.
**Bad:** "se puede, los modelos tienen 1M tokens".
**Decisión:** bad → degradar.

### Probe 3
**Pregunta:** "Dónde guardarías episodic memory de 100K eventos?"
**Good signals:** vector store, indexed by embedding + timestamp metadata.
**Bad:** "una tabla SQL".
**Decisión:** señal de explored.

---

## prompt-patterns

### Probe 1
**Pregunta:** "¿Qué es PTCF?"
**Good signals:** Persona-Task-Context-Format, explica los 4.
**Bad:** "no me acuerdo".
**Decisión:** ata.

### Probe 2
**Pregunta:** "CoT vs ToT — ¿cuándo cada uno?"
**Good signals:** CoT lineal, ToT explora ramas, ToT más caro para problemas con múltiples paths.
**Bad:** no distingue.
**Decisión:** ata.

### Probe 3
**Pregunta:** "Few-shot, ¿cuándo es overkill?"
**Good signals:** cuando el modelo es fuerte zero-shot, agrega tokens, puede bias hacia ejemplos.
**Bad:** "siempre más ejemplos mejor".
**Decisión:** señal de senior.

---

## chunking-strategy

### Probe 1
**Pregunta:** "¿Cuánto importa el chunking en RAG, de 1 a 10?"
**Good signals:** 9-10, explica que es la unidad de retrieval, afecta todo downstream.
**Bad:** "5, el modelo se adapta".
**Decisión:** señal clara de gap.

### Probe 2
**Pregunta:** "Fixed-size 500 tokens sin overlap, ¿qué problema tiene?"
**Good signals:** parte frases por la mitad, contexto cortado en bordes, embeddings malos.
**Bad:** "ninguno, es lo standard".
**Decisión:** bad → degradar.

### Probe 3
**Pregunta:** "PDFs con tablas — ¿cómo chunkeás?"
**Good signals:** layout-aware parsing primero, tablas como units atómicos, no partir.
**Bad:** "el splitter default".
**Decisión:** señal senior si lo nombra.

---

## embeddings

### Probe 1
**Pregunta:** "Si cambiás de modelo de embeddings, ¿qué hay que hacer?"
**Good signals:** re-embeddear TODO el corpus, re-indexar.
**Bad:** "nada, son comparables".
**Decisión:** crítico — bad → unknown.

### Probe 2
**Pregunta:** "Para 100M docs, ¿cómo bajás costo de embeddings?"
**Good signals:** Batch API, modelo más chico, dedupe, cache.
**Partial:** menciona uno.
**Bad:** "lo pago".
**Decisión:** ata.

### Probe 3
**Pregunta:** "¿Cuándo fine-tunear embeddings?"
**Good signals:** dominio muy específico con eval pobre + tenés pairs.
**Bad:** "siempre mejor".
**Decisión:** señal senior.

---

## vector-search

### Probe 1
**Pregunta:** "ANN vs exact search — tradeoff."
**Good signals:** ANN aproximado, escala bien, pierde recall un poco. Exact garantiza pero no escala.
**Bad:** confunde.
**Decisión:** ata.

### Probe 2
**Pregunta:** "Cosine vs dot product."
**Good signals:** equivalentes si vectores están normalizados. La mayoría de embeddings modernos lo están.
**Bad:** no sabe.
**Decisión:** señal de detail mastery.

### Probe 3
**Pregunta:** "¿Cuándo NO usar vector DB?"
**Good signals:** corpus chico (<10K), updates extremos, latencia ultra-baja con queries simples.
**Bad:** "siempre vector DB para RAG".
**Decisión:** señal senior.

---

## hybrid-retrieval

### Probe 1
**Pregunta:** "¿Qué problema resuelve hybrid (BM25 + semantic)?"
**Good signals:** BM25 mata en keywords exactos, semantic en paraphrasing. Cada uno cubre debilidad del otro.
**Bad:** "no sé / es más fuerte".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Reciprocal Rank Fusion — ¿qué es?"
**Good signals:** combina rankings por posición (1/(K+rank)), no por scores raw.
**Bad:** no lo conoce.
**Decisión:** practiced+ si lo sabe.

### Probe 3
**Pregunta:** "¿Cuándo hybrid es overkill?"
**Good signals:** queries uniformemente paraphrásticas, latencia tight, corpus tiny.
**Bad:** "siempre conviene".
**Decisión:** señal senior.

---

## re-ranking

### Probe 1
**Pregunta:** "¿Por qué retrieval + rerank en vez de retrieval solo?"
**Good signals:** retrieval prioriza recall (trae candidatos), rerank prioriza precision (ordena top-K).
**Bad:** "para tener más resultados".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Bi-encoder vs cross-encoder."
**Good signals:** bi=embeds separados (rápido, escalable), cross=joint scoring (preciso, no precomputable).
**Bad:** desconoce.
**Decisión:** señal practiced+.

### Probe 3
**Pregunta:** "Sweet spot de top-K para rerank."
**Good signals:** ~50 retrieve, ~10 después de rerank.
**Bad:** "no sé".
**Decisión:** ata.

---

## mcp-protocol

### Probe 1
**Pregunta:** "MCP vs function calling — diferencia conceptual."
**Good signals:** MCP = protocolo entre app y servers de tools (reutilizable). Function calling = entre LLM y app. MCP estandariza el server side.
**Bad:** "son lo mismo".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Riesgos de seguridad de MCP."
**Good signals:** stdio servers ejecutan código, prompt injection vía resources, privilege escalation.
**Bad:** "ninguno".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "3 primitivas core de MCP."
**Good signals:** Tools, Resources, Prompts.
**Bad:** no sabe.
**Decisión:** practiced+ si las nombra.

---

## async-patterns

### Probe 1
**Pregunta:** "¿Por qué async para LLM calls?"
**Good signals:** I/O bound, paralelizás llamadas independientes, throughput sube.
**Bad:** "más rápido el código".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Pattern para 100 LLM calls en paralelo respetando rate limit."
**Good signals:** asyncio.gather + semaphore, timeout per call, return_exceptions.
**Bad:** "for loop con await".
**Decisión:** crítico.

### Probe 3
**Pregunta:** "Diferencia asyncio vs threading para esto."
**Good signals:** asyncio cooperative single-thread, ideal I/O, escala mejor. Threading preemptive, GIL, más pesado.
**Bad:** confuso.
**Decisión:** ata.

---

## sse-streaming

### Probe 1
**Pregunta:** "SSE vs WebSocket — cuándo cada uno."
**Good signals:** SSE unidir (token stream), WebSocket bidir (interrupciones, collab).
**Bad:** "lo mismo".
**Decisión:** ata.

### Probe 2
**Pregunta:** "¿Por qué stremear tokens?"
**Good signals:** perceived latency, TTFT como métrica.
**Bad:** "es más rápido".
**Decisión:** practiced si dice TTFT.

### Probe 3
**Pregunta:** "Stream corta a la mitad en prod, ¿qué chequeás?"
**Good signals:** proxy timeouts, buffering CDN/nginx, worker timeout, network MTU.
**Bad:** "reinicio el servidor".
**Decisión:** señal senior si sabe el debugging.

---

## rate-limits

### Probe 1
**Pregunta:** "TPM y RPM, ¿qué son?"
**Good signals:** tokens y requests per minute, ambos limitados por provider.
**Bad:** confuso.
**Decisión:** ata.

### Probe 2
**Pregunta:** "Exponential backoff con jitter — ¿qué es?"
**Good signals:** delay crece exponencial (1,2,4,8) + random jitter para evitar thundering herd.
**Bad:** "retry cada 1 seg".
**Decisión:** señal explored+.

### Probe 3
**Pregunta:** "5 API keys del mismo provider, ¿cómo distribuís?"
**Good signals:** round-robin o least-loaded, failover en 429, NO mezclar keys de proyectos distintos.
**Bad:** "uso una".
**Decisión:** señal senior si menciona attribution.

---

## prompt-caching

### Probe 1
**Pregunta:** "Prompt caching de Anthropic, ¿cuánto baja el costo?"
**Good signals:** ~90% off en input tokens cached, 1.25x premium en write.
**Bad:** no sabe.
**Decisión:** ata.

### Probe 2
**Pregunta:** "Dónde poné cache_control."
**Good signals:** al final de bloque estable, antes de la parte dinámica.
**Bad:** "al principio".
**Decisión:** practiced si dice bien.

### Probe 3
**Pregunta:** "¿Cuándo prompt caching NO te ayuda?"
**Good signals:** cada request prompt único, prefix <1024 tokens, uso esporádico (TTL 5 min muere).
**Bad:** "siempre ayuda".
**Decisión:** señal senior.

---

## cost-optimization

### Probe 1
**Pregunta:** "5 técnicas para bajar costo LLM."
**Good signals:** caching, batch API, modelo chico, prompt trimming, response cache.
**Partial:** 2-3.
**Bad:** 1 o ninguna.
**Decisión:** ata.

### Probe 2
**Pregunta:** "Router pattern para cost — ¿qué es?"
**Good signals:** clasificador barato decide si usar modelo chico o grande.
**Bad:** no sabe.
**Decisión:** practiced+.

### Probe 3
**Pregunta:** "Agente cuesta 10 USD/sesión, querés 2 USD. ¿Por dónde empezás?"
**Good signals:** profile primero (tokens por step), atacá el más caro (system prompt o context), prompt caching.
**Bad:** "uso GPT-3.5".
**Decisión:** señal senior si metodológico.

---

## langchain-basics

### Probe 1
**Pregunta:** "¿Qué es LCEL?"
**Good signals:** LangChain Expression Language, declarative con `|`, runnables.
**Bad:** no sabe.
**Decisión:** ata.

### Probe 2
**Pregunta:** "¿Cuándo NO usar LangChain?"
**Good signals:** proyecto chico, abstracción ofusca, equipo prefiere SDK directo.
**Bad:** "siempre conviene LangChain".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Migrar de v0.0.x a v0.3.x — ¿qué cuidás?"
**Good signals:** breaking changes serios, eval baseline pre-migración, módulo por módulo.
**Bad:** "actualizo y veo".
**Decisión:** señal senior.

---

## langgraph-dags

### Probe 1
**Pregunta:** "LangGraph vs LangChain agents — qué resuelve."
**Good signals:** grafos de estado explícitos, checkpointing, HITL, debuggable, branching no trivial.
**Bad:** "lo mismo con otro nombre".
**Decisión:** ata.

### Probe 2
**Pregunta:** "¿Qué es un edge condicional?"
**Good signals:** función router(state)→next_node, branching dinámico.
**Bad:** confuso.
**Decisión:** practiced+.

### Probe 3
**Pregunta:** "Loop infinito en LangGraph, ¿cómo lo evitás?"
**Good signals:** recursion_limit, detección de ciclos en router, tracing.
**Bad:** "no debería pasar".
**Decisión:** señal senior.

---

## state-management

### Probe 1
**Pregunta:** "State in-memory vs persistido — tradeoff."
**Good signals:** in-mem pierde en restart, persistido permite resume/HITL/multi-instance.
**Bad:** confuso.
**Decisión:** ata.

### Probe 2
**Pregunta:** "State crece descontrolado, ¿qué hacés?"
**Good signals:** truncation, summarization, tiered (hot/warm/cold).
**Bad:** "compro RAM".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "State leak entre users, ¿causas comunes?"
**Good signals:** global vars, singleton agent, thread_id mal scope.
**Bad:** no sabe.
**Decisión:** señal senior si lo razona.

---

## checkpointing

### Probe 1
**Pregunta:** "¿Para qué sirve checkpointing en LangGraph?"
**Good signals:** resume crash, HITL pause/resume, time-travel debug, audit.
**Partial:** 1-2 usos.
**Bad:** desconoce.
**Decisión:** ata.

### Probe 2
**Pregunta:** "Thread_id en checkpointer — qué es."
**Good signals:** identifier de conversación, scopea state, best practice user_id:session_id.
**Bad:** confuso.
**Decisión:** practiced+.

### Probe 3
**Pregunta:** "Long-running agent debe sobrevivir pod restart, ¿cómo?"
**Good signals:** checkpoint after each node, durable backend (Postgres HA), idempotency en tools side-effect.
**Bad:** "retry desde cero".
**Decisión:** señal senior.

---

## llamaindex-vs-langchain

### Probe 1
**Pregunta:** "¿Cuándo LlamaIndex sobre LangChain?"
**Good signals:** retrieval-first, indexing complejo (graph/hierarchical), query engines avanzados.
**Bad:** "lo mismo".
**Decisión:** ata.

### Probe 2
**Pregunta:** "¿Se pueden combinar?"
**Good signals:** sí, LlamaIndex como retriever dentro de LangChain app.
**Bad:** "son rivales".
**Decisión:** practiced+.

### Probe 3
**Pregunta:** "Senior dice 'uso SDK directo, no framework' — ¿válido?"
**Good signals:** sí para scope chico/control crítico, tradeoff es reimplementar patterns.
**Bad:** "no, hay que usar framework".
**Decisión:** señal senior.

---

## supervisor-pattern

### Probe 1
**Pregunta:** "Supervisor vs single agent con muchas tools — ¿cuándo cada uno?"
**Good signals:** supervisor cuando workers con prompts muy distintos o necesitás modularidad/escala. Single cuando tools homogéneas, simple.
**Bad:** "siempre supervisor".
**Decisión:** ata.

### Probe 2
**Pregunta:** "God-object trap en supervisor — qué es."
**Good signals:** supervisor sabe demasiado de cada worker, prompt enorme, fix = solo conoce contratos.
**Bad:** no lo identifica.
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Supervisor loopea entre 2 workers, ¿causa?"
**Good signals:** workers no devuelven status complete claro, supervisor sin branch de exit.
**Bad:** "max iter".
**Decisión:** señal senior si va al root cause.

---

## hierarchical-pattern

### Probe 1
**Pregunta:** "¿Cuándo hierarchical en vez de single supervisor?"
**Good signals:** >10-15 workers, domain separation clara.
**Bad:** "siempre que se pueda".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Desventajas de hierarchical."
**Good signals:** latencia (más hops), costo, debugging, bottleneck en super-supervisor.
**Bad:** "ninguna".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Dependencias cross-team — ¿cómo las modelás?"
**Good signals:** shared state, message passing via super-supervisor, no llamadas directas worker-worker.
**Bad:** no sabe.
**Decisión:** señal senior.

---

## horizontal-network

### Probe 1
**Pregunta:** "Horizontal vs supervisor — ¿cuándo cada uno?"
**Good signals:** horizontal para emergent/colaborativo sin descomposición clara. Supervisor para determinismo + accountability.
**Bad:** "horizontal es siempre mejor".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Red horizontal nunca termina, ¿por qué?"
**Good signals:** sin coordinador falta "task complete" trigger. Soluciones: observer, voting done, time limit, convergence detection.
**Bad:** no sabe.
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "¿Cómo testeás horizontal con behavior impredecible?"
**Good signals:** multiples runs + variance, property-based, statistical asserts.
**Bad:** "no se puede".
**Decisión:** señal senior.

---

## task-delegation

### Probe 1
**Pregunta:** "Cómo decide supervisor a quién delegar — opciones."
**Good signals:** routing prompt LLM, classifier (embedding/fine-tune), rules, híbrido.
**Bad:** "el LLM elige".
**Decisión:** ata.

### Probe 2
**Pregunta:** "20 workers no entran en un prompt, ¿cómo routás?"
**Good signals:** two-stage routing, embedding-based, cache de routing decisions.
**Bad:** no sabe.
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Routing correcto pero workers fallan — ¿qué pasa?"
**Good signals:** misalignment task contract, worker no sabe qué success significa.
**Bad:** "los workers están rotos".
**Decisión:** señal senior si va al contract.

---

## conflict-resolution

### Probe 1
**Pregunta:** "3 estrategias de conflict resolution multi-agent."
**Good signals:** voting, arbiter, debate, HITL, priority.
**Partial:** 2.
**Bad:** "no sé".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Voting con N agentes — ¿cuándo es mala idea?"
**Good signals:** errores correlacionados (mismo modelo), expertise asimétrica, requires reasoning no consensus.
**Bad:** "siempre funciona".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Empate en voting — ¿qué hacés?"
**Good signals:** tie-breaker arbiter, role priority, HITL, conservative default. Diseño preventivo: N impar.
**Bad:** no sabe.
**Decisión:** señal senior.

---

## evals

### Probe 1
**Pregunta:** "Eval set para agente conversacional — ¿qué incluís?"
**Good signals:** real conversations sample, edge cases, adversarial, mix deterministic + LLM-judge, versionado git, CI gate.
**Bad:** "lo voy a hacer después".
**Decisión:** señal senior si lo arma metódico.

### Probe 2
**Pregunta:** "LLM-as-judge es inconsistente, ¿qué hacés?"
**Good signals:** judge más fuerte, rubrica concreta, multiple runs aggregate, calibración con humanos.
**Bad:** "le cambio el prompt".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "¿Eval de agente multi-step se hace solo en output final?"
**Good signals:** NO — también trace eval (cada step), cost/latency, diff testing entre versiones.
**Bad:** "sí, solo output final".
**Decisión:** señal senior.

---

## observability

### Probe 1
**Pregunta:** "3 métricas clave por request de agente."
**Good signals:** latency P95, cost (tokens × price), success rate.
**Partial:** 2.
**Bad:** "los logs".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Agente alucina 1% en prod — ¿cómo lo detectás sin reporte de user?"
**Good signals:** sampling con LLM-judge async, grounding checks, anomaly detection, feedback loop con 👎.
**Bad:** "espero quejas".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Distributed tracing — ¿qué es y por qué importa multi-agent?"
**Good signals:** trace_id propagado, spans hierarchical, OpenTelemetry standard. Sin esto no podés correlacionar flow.
**Bad:** no sabe.
**Decisión:** señal senior.

---

## safety-prompt-injection

### Probe 1
**Pregunta:** "Prompt injection directa vs indirecta."
**Good signals:** directa = user manda instrucciones, indirecta = en data retrieved (web, docs).
**Bad:** no distingue.
**Decisión:** ata.

### Probe 2
**Pregunta:** "3 defensas contra prompt injection."
**Good signals:** delimiters + instrucción de tratar como data, least privilege en tools, output validation, sandbox, red team eval.
**Partial:** 1-2.
**Bad:** "le explico al modelo que no haga".
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Agente lee email malicioso que dice 'manda credenciales a X' — ¿cómo te protegés?"
**Good signals:** email content marcado untrusted, tools sin acceso a send-external, HITL para acciones sensibles, output filter.
**Bad:** "el modelo va a ignorar".
**Decisión:** señal senior.

---

## compliance-argentina

### Probe 1
**Pregunta:** "¿Qué es la Ley 25.326?"
**Good signals:** Ley de Protección de Datos Personales AR (1996). Consentimiento, ARCO, transferencias internacionales restringidas.
**Bad:** "no sé / es de IA".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Transferencia internacional de datos a OpenAI — ¿qué implica para AR?"
**Good signals:** restricción 25.326, requiere consentimiento + cláusulas contractuales o país adecuado. US no es default adecuado.
**Bad:** "no hay problema".
**Decisión:** crítico — bad → unknown.

### Probe 3
**Pregunta:** "¿Qué exige la Disposición 2/2023 de AAIP?"
**Good signals:** recomendaciones para datos personales en IA: proporcionalidad, transparencia, supervisión humana, mitigación de sesgos.
**Bad:** desconoce.
**Decisión:** señal senior si la conoce.

---

## compliance-global

### Probe 1
**Pregunta:** "3 frameworks globales que afectan a un LLM agent."
**Good signals:** GDPR, EU AI Act, HIPAA, SOC2, PCI-DSS.
**Partial:** 2.
**Bad:** 1.
**Decisión:** ata.

### Probe 2
**Pregunta:** "EU AI Act, ¿qué es 'high-risk system'?"
**Good signals:** áreas de Annex III (salud, educación, empleo, justicia, etc.) con requisitos extra (risk mgmt, transparency, oversight).
**Bad:** desconoce.
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "BAA — ¿qué es y cuándo lo necesitás?"
**Good signals:** Business Associate Agreement, HIPAA, obligatorio si procesás PHI vía third-party LLM.
**Bad:** no sabe.
**Decisión:** señal senior si conoce US healthcare.

---

## cost-attribution

### Probe 1
**Pregunta:** "Cost attribution — ¿qué es y por qué importa?"
**Good signals:** atribuir tokens/costo a tenant/feature/user, importa para chargeback B2B, identificar features caras, prevenir abuse.
**Bad:** "ver la factura total".
**Decisión:** ata.

### Probe 2
**Pregunta:** "Implementación básica — ¿cómo lo armás?"
**Good signals:** tag metadata en cada call, capturar usage del response, sink async a OLAP, dashboards.
**Bad:** no sabe.
**Decisión:** señal senior.

### Probe 3
**Pregunta:** "Tenant excede budget overnight por agent runaway — post-mortem."
**Good signals:** root cause (loop sin max_iter o atack), kill switch automático, alerting a 50% no 100%, transparency dashboard al tenant.
**Bad:** "le cobro extra".
**Decisión:** señal senior.

---

## Decisión post-respuesta — reglas generales

| Resultado del probe | Acción de mastery |
|---|---|
| Good + estaba unknown | Upgrade tentativo a explored (con flag pending evidence) |
| Good + estaba explored/practiced | Mantener nivel |
| Good + estaba mastered | Mantener |
| Partial + estaba practiced/mastered | Considerar degradar (con aviso + veto) |
| Partial + estaba unknown/explored | Mantener |
| Bad + estaba practiced/mastered | Degradar 1 nivel (con aviso + veto) |
| Bad + estaba unknown/explored | Forzar unknown |

Después de cualquier cambio: `mem_save` con topic_key `skill/ai-engineer-mentor/mastery/{concept}` + nota en `history[]`.
