# External References — Fuentes oficiales curadas

**Cuándo se carga**: lazy, cuando el mentor necesita pointar a docs canónicos para un gap del libro.

**Cómo usar**: ordenado por tema/concepto. Cada entrada tiene URL real (NUNCA inventada), qué cubre, qué tenés que entender de ahí, y cuándo volver.

**Regla**: si no estás seguro del link exacto, escribí "buscar en docs.{provider}.com/{topic}" en vez de inventar.

---

## Tool use / Function calling

### Anthropic Tool Use Docs
**Tipo:** docs oficiales
**URL:** https://docs.anthropic.com/en/docs/build-with-claude/tool-use
**Qué cubre:** definición de tools, schema, parallel tool use, tool_choice, ejemplos en Python/TS.
**Qué tenés que entender:**
- Diferencia entre `tool_choice: auto / any / tool / none`
- Cómo Claude devuelve `tool_use` blocks en `content`
- Parallel tool use cuándo y cómo
- Pricing impact (tool calls = tokens)
**Tiempo de lectura:** 45 min
**Cuándo volver:** cada vez que diseñes tools nuevas o debuggees un comportamiento raro de tool selection.

### OpenAI Function Calling / Structured Outputs
**Tipo:** docs oficiales
**URL:** https://platform.openai.com/docs/guides/function-calling y https://platform.openai.com/docs/guides/structured-outputs
**Qué cubre:** function calling clásico + structured outputs strict mode (JSON Schema constrained generation).
**Qué tenés que entender:**
- `strict: true` cambia comportamiento (no extra fields, types enforced at decode time)
- Limitaciones de schemas soportados (no $ref complex, etc.)
- Diferencia entre `tools` y `response_format`
- Parallel function calling
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al elegir entre structured outputs vs function calling, o cuando un schema falla en strict mode.

---

## MCP (Model Context Protocol)

### Anthropic MCP Specification
**Tipo:** spec oficial
**URL:** https://modelcontextprotocol.io/specification
**Qué cubre:** protocolo completo, primitivas (tools, resources, prompts, sampling), transport (stdio, HTTP), security model.
**Qué tenés que entender:**
- Las 3 primitivas core: Tools, Resources, Prompts
- Cliente-server architecture (host app es cliente, servers exponen capabilities)
- Capability negotiation
- Sampling pattern (server pide al cliente que llame LLM)
- Security implications (especialmente stdio servers)
**Tiempo de lectura:** 2 horas (spec entera) o 30 min (overview)
**Cuándo volver:** antes de escribir un MCP server, o cuando integres tu agente con un server externo.

### MCP Reference Servers (GitHub)
**Tipo:** repos de referencia
**URL:** https://github.com/modelcontextprotocol/servers
**Qué cubre:** implementaciones de referencia (filesystem, git, GitHub, Slack, etc.) en Python y TypeScript.
**Qué tenés que entender:**
- Patterns para exponer resources (URI schemes)
- Tool naming conventions
- Error handling estándar
- Auth patterns por server type
**Tiempo de lectura:** 1 hora (skim) por server relevante
**Cuándo volver:** como starter template para tu propio MCP server.

---

## Prompt Caching

### Anthropic Prompt Caching Docs
**Tipo:** docs oficiales
**URL:** https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching
**Qué cubre:** cómo usar `cache_control`, qué se puede cachear, TTL, pricing.
**Qué tenés que entender:**
- TTL 5 min default (configurable 1h con beta header)
- Write cost 1.25x, read cost 0.1x
- Min 1024 tokens para cachear (Sonnet/Opus) / 2048 (Haiku)
- Cache breakpoints — hasta 4 por request
- Cómo medir hit rate via `usage.cache_creation_input_tokens` y `usage.cache_read_input_tokens`
**Tiempo de lectura:** 30 min
**Cuándo volver:** al diseñar estructura del prompt para maximizar cacheable prefix.

### OpenAI Prompt Caching (automático)
**Tipo:** docs oficiales
**URL:** https://platform.openai.com/docs/guides/prompt-caching
**Qué cubre:** caching automático para prompts >1024 tokens, sin necesidad de markers.
**Qué tenés que entender:**
- Automático: no necesitás cache_control
- 50% off en cached input (vs 90% de Anthropic)
- Eviction policy desconocida (best effort)
- Aplica a prefix exacto
**Tiempo de lectura:** 15 min
**Cuándo volver:** comparar economics con Anthropic, al planear costos.

---

## Frameworks de orquestación

### LangChain Docs (overview)
**Tipo:** docs oficiales
**URL:** https://python.langchain.com/docs/introduction/ (Python) / https://js.langchain.com (JS)
**Qué cubre:** LCEL, chains, agents, retrievers, memory, tools, providers.
**Qué tenés que entender:**
- LCEL (`|` operator, runnables)
- Diferencia entre v0.0.x, v0.1+, v0.3+ (breaking changes serios)
- `Runnable` interface: invoke / stream / batch / async variants
- Integraciones core (OpenAI, Anthropic, vector stores)
**Tiempo de lectura:** 3 horas (overview) / días (deep)
**Cuándo volver:** al elegir abstraction o debuggear comportamiento.

### LangGraph Docs
**Tipo:** docs oficiales
**URL:** https://langchain-ai.github.io/langgraph/
**Qué cubre:** state graphs, nodes, edges, checkpointers, HITL, multi-agent patterns.
**Qué tenés que entender:**
- State y reducers (especialmente `add_messages`)
- Conditional edges con routers
- Checkpointer interface (in-memory, SQLite, Postgres)
- `interrupt` para HITL
- Subgraphs y multi-agent topologies
- Time-travel y state inspection
**Tiempo de lectura:** 4 horas para cubrir bien
**Cuándo volver:** al diseñar cualquier agente multi-step en LangGraph.

### LlamaIndex Docs
**Tipo:** docs oficiales
**URL:** https://docs.llamaindex.ai/
**Qué cubre:** indexing, retrievers, query engines, agents, evaluation, observability.
**Qué tenés que entender:**
- Diferencia entre Index types (VectorStoreIndex, KnowledgeGraphIndex, ListIndex)
- Query Engines avanzados: SubQuestion, Router, Recursive
- Node parsers (chunking) y postprocessors (rerank)
- LlamaParse para PDFs/layouts
- Evaluation con LLM judges
**Tiempo de lectura:** 4 horas
**Cuándo volver:** cuando RAG es central y necesitás abstractions más allá de "embed + search".

---

## Observability / Evals

### Langfuse Docs
**Tipo:** docs oficiales (open-source)
**URL:** https://langfuse.com/docs
**Qué cubre:** tracing, prompt management, evals, datasets, self-hosting.
**Qué tenés que entender:**
- SDK decorators (`@observe`)
- Trace / span / generation structure
- Prompt versioning y A/B testing
- Self-host con Docker (Postgres + ClickHouse)
- Integration con LangChain/LangGraph/OpenAI/Anthropic
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al instrumentar tu primer proyecto, o al migrar de LangSmith.

### LangSmith Docs
**Tipo:** docs oficiales
**URL:** https://docs.smith.langchain.com/
**Qué cubre:** tracing, evaluation, prompt management, datasets, integration con LangChain.
**Qué tenés que entender:**
- Trace UI (tree de spans, inputs/outputs por nodo)
- Eval framework con custom evaluators
- Dataset management y annotations
- Comparison runs (regression testing)
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al evaluar baseline o regression entre versiones.

### Promptfoo
**Tipo:** docs oficiales (CLI tool, open-source)
**URL:** https://www.promptfoo.dev/docs/intro
**Qué cubre:** declarative YAML config para eval suites, CI-friendly, multi-provider.
**Qué tenés que entender:**
- YAML schema para `providers`, `prompts`, `tests`
- Asserts (equality, regex, LLM-judge, javascript)
- CI integration (GitHub Actions templates)
- Adversarial testing (red-team plugin)
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al setup CI gates para prompts/agentes.

---

## Vector DBs

### pgvector (Postgres extension)
**Tipo:** repo oficial + docs
**URL:** https://github.com/pgvector/pgvector
**Qué cubre:** instalación, índices (HNSW, IVFFlat), queries, performance tuning.
**Qué tenés que entender:**
- HNSW vs IVFFlat tradeoffs
- Parámetros HNSW (m, ef_construction, ef_search)
- Operadores: `<->` (L2), `<=>` (cosine), `<#>` (inner product)
- Combinación con filters SQL
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al tunear performance o decidir entre índices.

### Qdrant Docs
**Tipo:** docs oficiales
**URL:** https://qdrant.tech/documentation/
**Qué cubre:** collections, payload schema, filtering, hybrid search, scaling.
**Qué tenés que entender:**
- Payload (metadata) indexing
- Filtering pre vs post (Qdrant hace pre-filter eficiente)
- Hybrid search (sparse + dense)
- Multi-tenant patterns (collections vs filters)
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al diseñar multi-tenant o queries complejas con filters.

### Weaviate Docs
**Tipo:** docs oficiales
**URL:** https://weaviate.io/developers/weaviate
**Qué cubre:** schema, hybrid search built-in, GraphQL queries, multi-modal.
**Qué tenés que entender:**
- Schema-first (definís classes con properties)
- Hybrid search (alpha parameter mezcla BM25 + vector)
- Modules (vectorizers nativos, generative)
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al evaluar Weaviate vs Qdrant, especialmente para hybrid out-of-box.

---

## Re-ranking

### bge-reranker-v2-m3 (HuggingFace model card)
**Tipo:** model card
**URL:** https://huggingface.co/BAAI/bge-reranker-v2-m3
**Qué cubre:** modelo open-source multilingual de re-ranking, código de uso, benchmarks.
**Qué tenés que entender:**
- Multilingual (incluye español)
- Inference rápida en GPU, aceptable en CPU para low-throughput
- Score normalization (sigmoid)
- Fine-tunable con tu data
**Tiempo de lectura:** 30 min
**Cuándo volver:** al self-hostear re-ranker.

### Cohere Rerank
**Tipo:** docs API
**URL:** https://docs.cohere.com/docs/rerank-overview
**Qué cubre:** Rerank API, modelos (rerank-english-v3, rerank-multilingual-v3), pricing.
**Qué tenés que entender:**
- Pricing por call (~ $0.001-$0.002)
- Latencia típica (100-300ms)
- Max docs por call y max length
- Cuándo conviene vs self-hosted
**Tiempo de lectura:** 20 min
**Cuándo volver:** al elegir managed vs self-host de re-ranker.

---

## Compliance

### Argentina — AAIP
**Tipo:** autoridad regulatoria, normativas
**URL:** https://www.argentina.gob.ar/aaip
**Qué cubre:** Ley 25.326 Protección de Datos Personales, normativas relacionadas (Disposiciones), guías.
**Qué tenés que entender:**
- Ley 25.326 obligaciones (registro de tratamientos, consentimiento, ARCO)
- Disposición 2/2023 (recomendaciones IA y datos personales)
- Decreto 836/2024 (actualizaciones regulatorias recientes)
- Transferencias internacionales (Anexo I, países adecuados)
**Tiempo de lectura:** 4 horas para cubrir lo central
**Cuándo volver:** al diseñar compliance de cualquier app que toque datos personales de argentinos.

### EU AI Act (oficial)
**Tipo:** texto legal + resúmenes
**URL:** https://artificialintelligenceact.eu/ (resumen) y https://eur-lex.europa.eu (texto oficial)
**Qué cubre:** clasificación de sistemas (prohibited / high-risk / limited / minimal), obligaciones por tier, timeline de entrada en vigencia.
**Qué tenés que entender:**
- Annex III list de high-risk use cases
- Obligaciones high-risk (risk management, data quality, transparency, human oversight, technical docs, post-market monitoring)
- GPAI (General Purpose AI) obligations
- Fines hasta 7% revenue global
- Timeline: prohibited Feb 2025, GPAI Aug 2025, high-risk Aug 2026
**Tiempo de lectura:** 6 horas (resumen) / días (texto oficial)
**Cuándo volver:** antes de cualquier lanzamiento en EU, sobre todo high-risk.

### NIST AI Risk Management Framework
**Tipo:** framework
**URL:** https://www.nist.gov/itl/ai-risk-management-framework
**Qué cubre:** framework voluntario US para gestión de riesgo de IA, 4 funciones (Govern, Map, Measure, Manage).
**Qué tenés que entender:**
- 4 functions: Govern (cultura), Map (contexto), Measure (analytics), Manage (response)
- Profile concept (industry / use case specific)
- Aplica como buen baseline aunque no sea ley
**Tiempo de lectura:** 4 horas
**Cuándo volver:** para diseño de risk management en US, o como buena referencia general.

### ISO/IEC 42001 — AI Management System
**Tipo:** standard
**URL:** https://www.iso.org/standard/81230.html
**Qué cubre:** standard para AI Management Systems (analog a ISO 27001 para security).
**Qué tenés que entender:**
- Marco para AIMS (AI Management System)
- Requirements para policies, risk assessment, controls
- Certificación posible (enterprise sales lo piden)
**Tiempo de lectura:** Standard es paid; resúmenes gratuitos en blogs (Bureau Veritas, TÜV, etc.)
**Cuándo volver:** al targetear enterprise EU que exige certificación.

### OWASP Top 10 for LLM Applications
**Tipo:** guía community
**URL:** https://genai.owasp.org/
**Qué cubre:** top 10 vulnerabilidades específicas de apps LLM, con ejemplos y mitigaciones.
**Qué tenés que entender:**
- LLM01 Prompt Injection (direct & indirect)
- LLM02 Insecure Output Handling
- LLM03 Training Data Poisoning
- LLM06 Sensitive Information Disclosure
- LLM07 Insecure Plugin Design
- LLM08 Excessive Agency
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al hacer security review, especialmente con tools y agents.

---

## Papers seminales

### ReAct (Reason + Act)
**Tipo:** paper académico
**URL:** https://arxiv.org/abs/2210.03629
**Qué cubre:** introducción original del patrón ReAct, evaluación en benchmarks.
**Qué tenés que entender:**
- Motivation: combinar CoT con action en environments
- Comparación contra act-only y reason-only
- Failure modes documentados
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al estudiar la base de los agent loops modernos.

### RAG (Retrieval-Augmented Generation)
**Tipo:** paper seminal
**URL:** https://arxiv.org/abs/2005.11401
**Qué cubre:** introducción de RAG (Lewis et al. 2020), arquitectura DPR + seq2seq.
**Qué tenés que entender:**
- Diferencia entre RAG-Sequence y RAG-Token
- Por qué retrieval mejora factualidad
- Base conceptual de todo lo que vino después
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al explicar RAG de cero o defender la elección arquitectónica.

### DPO (Direct Preference Optimization)
**Tipo:** paper
**URL:** https://arxiv.org/abs/2305.18290
**Qué cubre:** alternativa a RLHF, más simple y estable, para fine-tuning con preferencias.
**Qué tenés que entender:**
- Bypass de la separate reward model (training directo con preferences)
- Cuándo conviene DPO vs RLHF tradicional
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al considerar fine-tuning con preferencias humanas.

### Lost in the Middle (long context)
**Tipo:** paper
**URL:** https://arxiv.org/abs/2307.03172
**Qué cubre:** evidencia empírica de que info en el medio de contextos largos se pierde más que en bordes.
**Qué tenés que entender:**
- Posición de info importa: top y bottom recall mejor
- Implica orden de chunks en RAG (ranking matters)
**Tiempo de lectura:** 30 min
**Cuándo volver:** al diseñar prompt structure y ordering de RAG context.

---

## Eval / Cookbooks

### OpenAI Evals Repo
**Tipo:** repo + framework
**URL:** https://github.com/openai/evals
**Qué cubre:** framework de evals (model-graded, multiple choice, etc.), templates, ejemplos.
**Qué tenés que entender:**
- YAML config para registrar evals
- Custom graders (Python)
- Cómo correr contra distintos modelos
**Tiempo de lectura:** 2 horas (incluyendo setup local)
**Cuándo volver:** como referencia para diseño de evals.

### Anthropic Cookbook
**Tipo:** repo de recetas
**URL:** https://github.com/anthropics/anthropic-cookbook
**Qué cubre:** ejemplos prácticos: classification, RAG, tool use, evaluation, prompt engineering.
**Qué tenés que entender:**
- Patterns recomendados por Anthropic
- Notebooks con código corredor
- Templates de evaluation
**Tiempo de lectura:** 30 min por receta relevante
**Cuándo volver:** al implementar un pattern nuevo con Claude.

---

## Rate limits / providers

### Anthropic Rate Limits
**Tipo:** docs
**URL:** https://docs.anthropic.com/en/api/rate-limits
**Qué cubre:** tiers, RPM/TPM por modelo, cómo subir de tier, manejo de 429.
**Qué tenés que entender:**
- Tier system (build → scale → enterprise)
- Separate limits per model
- Headers en response (`retry-after`, etc.)
**Tiempo de lectura:** 15 min
**Cuándo volver:** al planear escala o debug 429s.

### OpenAI Rate Limits
**Tipo:** docs
**URL:** https://platform.openai.com/docs/guides/rate-limits
**Qué cubre:** tiers, RPM/TPM/RPD por modelo, cómo subir, retry guide.
**Qué tenés que entender:**
- Tier upgrade flow (usage history + payment)
- Project-level limits
- Separate batch API limits
**Tiempo de lectura:** 15 min
**Cuándo volver:** al planear ramp-up o multi-key strategy.

---

## Batch APIs

### OpenAI Batch API
**Tipo:** docs
**URL:** https://platform.openai.com/docs/guides/batch
**Qué cubre:** 50% off, 24h SLA, format JSONL, status polling.
**Qué tenés que entender:**
- File upload (JSONL con `custom_id`)
- Polling job status
- Use cases ideales (evals, embedding bulk, async generation)
**Tiempo de lectura:** 30 min
**Cuándo volver:** al procesar workloads grandes async.

### Anthropic Message Batches API
**Tipo:** docs
**URL:** https://docs.anthropic.com/en/api/creating-message-batches
**Qué cubre:** equivalente Anthropic, 50% off, 24h SLA.
**Qué tenés que entender:**
- Hasta 100K messages per batch
- Polling vs webhooks
- Combinable con prompt caching
**Tiempo de lectura:** 30 min
**Cuándo volver:** al optimizar costo con workloads non-realtime.

---

## Streaming

### OpenAI Streaming Reference
**Tipo:** docs API
**URL:** https://platform.openai.com/docs/api-reference/streaming
**Qué cubre:** SSE format, chunks, usage handling.
**Qué tenés que entender:**
- Cada chunk es un delta
- Último chunk tiene `[DONE]`
- `stream_options: { include_usage: true }` para cost
**Tiempo de lectura:** 20 min
**Cuándo volver:** al implementar streaming endpoint.

### Anthropic Streaming
**Tipo:** docs API
**URL:** https://docs.anthropic.com/en/api/messages-streaming
**Qué cubre:** event types (message_start, content_block_delta, message_delta, message_stop).
**Qué tenés que entender:**
- Event-based (no solo deltas planos)
- Tool use streaming (input_json_delta)
- Usage en message_delta final
**Tiempo de lectura:** 30 min
**Cuándo volver:** al diferenciar de OpenAI streaming pattern.

---

## 🌐 Comunidades (Wisdom layer)

Acá está la pata que faltaba, loco. El conocimiento (los docs de arriba) y los ejercicios (los modos/milestones) NO alcanzan: la sabiduría real viene de probarte en el mundo, fuera del entorno seguro de aprendizaje. Cuando un concepto te llega a `mastered` o terminás un `project`, andá a una de estas comunidades a que te lo ROMPAN — gente más capa que vos te va a marcar lo que no ves. Esa fricción es el aprendizaje de verdad.

**Regla de oro (igual que el resto del file):** ningún link inventado. Si no estoy 100% seguro del URL exacto, te dejo la pista de búsqueda en vez de mandarte a una página fantasma. Confiar en un link falso destruye la confianza — y la confianza es todo.

---

### Generales / cross-hito

### r/LocalLLaMA
**Dónde:** Reddit
**Link:** https://www.reddit.com/r/LocalLLaMA/
**Para qué sirve:** open models, RAG casero, fine-tuning, deployment local, cuantización. Cross-hito, pero brilla en Hito 2 (retrieval) y Hito 3 (costos/self-host). Una de las comunidades más activas y prácticas del ecosistema.
**Reputación:** gente que corre modelos en su propio hardware, ingenieros que comparten benchmarks reales, no marketing. Nivel medio-alto, muy hands-on.
**Cómo aportar (no solo consumir):** posteá tu setup con números reales (tokens/seg, VRAM, costo), o respondé a alguien que está peleando con un problema que vos ya resolviste.

### Latent Space
**Dónde:** Discord + podcast (swyx & team)
**Link:** https://www.latent.space/ (desde ahí el link al Discord)
**Para qué sirve:** AI engineering de frontera — lo que viene, entrevistas a builders top, discusión de papers y tooling nuevo. Cross-hito, fuerte en visión de producto/arquitectura.
**Reputación:** swyx acuñó el término "AI Engineer". Está la gente que está definiendo el campo. Nivel alto.
**Cómo aportar (no solo consumir):** compartí un writeup de algo que construiste, o sumá una perspectiva en las discusiones de episodios — ahí se nota quién piensa.

### Hacker News
**Dónde:** foro (Y Combinator)
**Link:** https://news.ycombinator.com/
**Para qué sirve:** discusión técnica de alto nivel sobre todo lo que sale en IA (y software en general). Cross-hito. Ideal para calibrar señal vs hype.
**Reputación:** founders, ingenieros senior, autores de las herramientas que usás. Los comentarios suelen valer más que el artículo.
**Cómo aportar (no solo consumir):** publicá tu proyecto en "Show HN" y bancá las críticas, o comentá con experiencia técnica real (no opinión vacía — ahí te comen).

### r/MachineLearning
**Dónde:** Reddit
**Link:** https://www.reddit.com/r/MachineLearning/
**Para qué sirve:** research-leaning — papers, métodos, discusión académica. Útil de fondo para todos los hitos, sobre todo cuando querés entender el "por qué" detrás de una técnica.
**Reputación:** investigadores y PhD students. Más teórica que r/LocalLLaMA. Nivel alto en research.
**Cómo aportar (no solo consumir):** en los hilos semanales ("Simple Questions") preguntá bien, y si reproducís un paper compartí los resultados.

### Hugging Face — Forums & Discord
**Dónde:** foro oficial + Discord
**Link:** https://discuss.huggingface.co/ (foro) — Discord: buscar el invite desde https://huggingface.co/ (footer/community)
**Para qué sirve:** models, embeddings, fine-tuning, datasets, `transformers`/`sentence-transformers`. Fuerte en Hito 2 (embeddings) y cualquier cosa de open models.
**Reputación:** maintainers de las libs y autores de modelos andan dando vueltas. Nivel alto en lo práctico de modelos.
**Cómo aportar (no solo consumir):** subí un model card o dataset propio, o respondé issues de gente que arranca con `transformers`.

---

### Por hito

### Hito 1 — Fundamentos (tool-use, prompting, JSON mode)

### Anthropic Developer Community (Discord)
**Dónde:** Discord oficial de Anthropic
**Link:** buscar el invite desde https://www.anthropic.com/ o desde https://docs.anthropic.com/ (sección community) — NO te paso un invite directo porque rotan
**Para qué sirve:** tool use, prompt engineering, structured output con Claude. Justo el corazón del Hito 1.
**Reputación:** devs de Anthropic responden, y hay builders serios. Nivel medio-alto.
**Cómo aportar (no solo consumir):** compartí un patrón de prompting que te funcionó con métricas, o ayudá a alguien a debuggear tool selection.

### OpenAI Developer Community
**Dónde:** foro oficial
**Link:** https://community.openai.com/
**Para qué sirve:** function calling, structured outputs, Assistants/Responses API. Complementa el Hito 1.
**Reputación:** comunidad enorme, calidad variable, pero los hilos técnicos buenos son oro. Staff de OpenAI participa.
**Cómo aportar (no solo consumir):** documentá un workaround a una limitación de la API — eso ayuda a miles.

### Hito 2 — Retrieval & MCP

### MCP Community (GitHub Discussions)
**Dónde:** GitHub Discussions del org oficial
**Link:** https://github.com/modelcontextprotocol (ver discussions/issues en los repos del org)
**Para qué sirve:** todo lo de Model Context Protocol — diseño de servers, primitivas, transport. Núcleo del concepto `mcp-protocol`.
**Reputación:** ahí está la gente que define el protocolo. Nivel alto.
**Cómo aportar (no solo consumir):** abrí un MCP server propio y compartilo, o reportá un edge case de la spec con repro.

### LlamaIndex Discord
**Dónde:** Discord oficial
**Link:** buscar el invite desde https://www.llamaindex.ai/ (footer "Community")
**Para qué sirve:** RAG serio — chunking, retrievers, query engines, evaluation de retrieval. Hito 2 puro (y toca Hito 4).
**Reputación:** maintainers activos, gente con pipelines RAG en producción. Nivel medio-alto.
**Cómo aportar (no solo consumir):** compartí tu estrategia de chunking con resultados de eval, o respondé dudas de retrieval.

### Vector DB communities — Qdrant & Weaviate
**Dónde:** Discord oficial de cada uno
**Link:** Qdrant: buscar invite desde https://qdrant.tech/ — Weaviate: buscar invite desde https://weaviate.io/community
**Para qué sirve:** tuning de índices (HNSW/IVFFlat), hybrid search, filtering, multi-tenant. Para cuando `vector-search` y `hybrid-retrieval` se ponen serios.
**Reputación:** ingenieros de las DBs responden. Nivel alto en lo específico.
**Cómo aportar (no solo consumir):** posteá benchmarks de tu colección o un patrón de filtering que descubriste.

> Nota: para RAG general también buscá r/Rag en Reddit (buscar en reddit.com: "Rag" — verificá que el sub esté activo antes de confiar).

### Hito 3 — Async & Costos

### r/LLMDevs
**Dónde:** Reddit
**Link:** https://www.reddit.com/r/LLMDevs/
**Para qué sirve:** developers construyendo con LLMs — costos, latencia, rate limits, arquitectura de APIs. Buen fit para Hito 3.
**Reputación:** developers prácticos, menos hype que otros subs. Nivel medio.
**Cómo aportar (no solo consumir):** compartí tu estrategia de routing por costo con números, o cómo manejás backpressure.

> Para detalle fino de rate limits/batch, los **foros de cada provider** (Anthropic Discord, OpenAI Community de arriba) son el mejor canal — ahí responde quien conoce los límites reales.

### Hito 4 — Frameworks (orquestación)

### LangChain & LangGraph (Discord + GitHub)
**Dónde:** Discord oficial + GitHub Discussions
**Link:** buscar el invite del Discord desde https://www.langchain.com/ (footer "Community") — GitHub: https://github.com/langchain-ai/langgraph/discussions
**Para qué sirve:** chains, LCEL, state graphs, checkpointers, HITL. Núcleo de Hito 4 (`langchain-basics`, `langgraph-dags`, `state-management`, `checkpointing`).
**Reputación:** maintainers muy activos en GitHub Discussions. Nivel medio-alto.
**Cómo aportar (no solo consumir):** reportá un bug con repro mínimo (vale oro), o compartí un grafo no trivial que armaste.

### LlamaIndex Discord
**Dónde:** Discord oficial (ya listado en Hito 2)
**Link:** desde https://www.llamaindex.ai/
**Para qué sirve:** acá también aplica para `llamaindex-vs-langchain` — cuándo conviene retrieval-first vs orquestación general.
**Reputación:** ver arriba.
**Cómo aportar (no solo consumir):** documentá una comparación honesta LlamaIndex vs LangChain en tu caso de uso.

### Hito 5 — Multi-Agente

### CrewAI (Community)
**Dónde:** foro/community oficial + GitHub
**Link:** buscar desde https://www.crewai.com/ (sección community) — GitHub: https://github.com/crewAIInc/crewAI (discussions)
**Para qué sirve:** orquestación de equipos de agentes con roles. Fit directo a `supervisor-pattern` y `task-delegation`.
**Reputación:** comunidad grande y creciente alrededor del framework. Nivel medio.
**Cómo aportar (no solo consumir):** compartí una crew que resuelve algo real y los failure modes que encontraste.

### Microsoft AutoGen (GitHub Discussions)
**Dónde:** GitHub Discussions del repo oficial
**Link:** https://github.com/microsoft/autogen (discussions)
**Para qué sirve:** conversación multi-agente, patrones peer-to-peer. Toca `horizontal-network` y `conflict-resolution`.
**Reputación:** equipo de Microsoft Research detrás, discusión técnica seria. Nivel alto.
**Cómo aportar (no solo consumir):** abrí una discussion con un patrón de coordinación que probaste, o reproducí un ejemplo y reportá qué falló.

### Hito 6 — Producción / Eval / Compliance

### Langfuse Discord
**Dónde:** Discord oficial (open-source)
**Link:** buscar el invite desde https://langfuse.com/ (footer "Community")
**Para qué sirve:** tracing, evals, cost tracking, self-hosting. Directo a `observability` y `cost-attribution`.
**Reputación:** maintainers responden rápido, gente instrumentando producción. Nivel medio-alto.
**Cómo aportar (no solo consumir):** compartí tu setup de eval/dataset, o un dashboard de costo por tenant que armaste.

### Weights & Biases Community
**Dónde:** comunidad oficial (foro + Discord)
**Link:** buscar desde https://wandb.ai/ (sección community) — foro: https://community.wandb.ai/
**Para qué sirve:** experiment tracking, evals, MLOps. Buen fit para `evals` y observabilidad de fondo.
**Reputación:** comunidad ML/MLOps madura. Nivel medio-alto.
**Cómo aportar (no solo consumir):** publicá un report reproducible de tus evals.

### MLOps Community
**Dónde:** Slack + podcast + eventos
**Link:** https://mlops.community/ (desde ahí el invite al Slack)
**Para qué sirve:** todo producción — deployment, monitoring, eval en el mundo real, compliance práctico. Cross Hito 6.
**Reputación:** una de las comunidades MLOps más grandes y serias. Practitioners de empresas reales. Nivel alto.
**Cómo aportar (no solo consumir):** contá un postmortem de algo que se rompió en prod — eso es ORO para la comunidad.

### DAIR.AI
**Dónde:** comunidad + newsletter + recursos
**Link:** https://github.com/dair-ai (org con guías, ej. Prompt Engineering Guide) — newsletter desde https://nlp.elvissaravia.com/
**Para qué sirve:** prompt engineering, papers curados, evaluación. Apoyo conceptual transversal, fuerte en eval y fundamentos.
**Reputación:** Elvis Saravia y comunidad NLP. Nivel medio-alto, muy didáctico.
**Cómo aportar (no solo consumir):** contribuí a las guías open-source o compartí un resumen de paper bien hecho.

---

### LATAM / Argentina (tu mercado local)

> Acá soy honesto: los links exactos de meetups rotan y muchos viven en plataformas distintas. NO te invento URLs. Te dejo las pistas de búsqueda — verificá vos cuál está vivo hoy.

### Comunidades AI/ML de Argentina
**Dónde:** Meetup + grupos de Telegram/Discord locales
**Link:** buscar en meetup.com: "Buenos Aires AI" y "Machine Learning Argentina" — verificá fecha del último evento antes de sumarte
**Para qué sirve:** networking local, charlas, contexto del mercado argentino. Cross-hito + contexto de empleo/freelance local.
**Reputación:** depende del grupo y del momento — chequeá actividad reciente. Mezcla de juniors y seniors.
**Cómo aportar (no solo consumir):** ofrecé dar una charla corta de un proyecto tuyo. Hablar en un meetup local te posiciona rápido.

### Python Argentina (PyAr)
**Dónde:** comunidad histórica (lista, eventos, redes)
**Link:** https://www.python.org.ar/
**Para qué sirve:** base sólida de Python en Argentina — el stack de casi todo lo de IA. Útil para fundamentos y networking local.
**Reputación:** comunidad veterana y respetada en el país. Nivel mixto, mucha gente capa.
**Cómo aportar (no solo consumir):** participá en PyCon Argentina o en los encuentros, presentá algo de agentes.

### Comunidades de IA en español
**Dónde:** Discord/Telegram varios + LinkedIn
**Link:** buscar en meetup.com: "inteligencia artificial" + tu ciudad — y seguir referentes de IA en LATAM en LinkedIn/X para encontrar sus comunidades
**Para qué sirve:** discutir en tu idioma, contexto regional, oportunidades LATAM.
**Reputación:** variable — aplicá el mismo filtro de reputación que arriba antes de invertir tiempo.
**Cómo aportar (no solo consumir):** traducí/explicá un concepto difícil en español — hay hambre de buen contenido técnico en castellano.

---

### Cómo usar esto (postura del mentor)

- **No te sumes a 10 comunidades.** Te perdés y no aportás en ninguna. Elegí DOS: una general (r/LocalLLaMA o Latent Space) y una del hito que estás laburando AHORA. Punto.
- **Wisdom = aportar, no solo leer.** Lurkear no te da sabiduría, te da la ilusión de aprender. Posteá tu proyecto, respondé preguntas de otros, mostrá tu código y bancá las críticas. La fricción es el aprendizaje.
- **El trigger es claro:** cuando el mentor te marca un concepto como `mastered`, o cuando terminás un `project` — ESE es el momento de ir a la comunidad a que te lo rompan. No antes (no tenés nada que mostrar), no después (perdés el momentum).
