# Tradeoffs Playbook — Decisiones técnicas de AI Engineering

**Cuándo se carga**: lazy, cuando el usuario pregunta "¿qué conviene entre X y Y?" o cuando un modo (interview/project/review) necesita comparar opciones.

**Cómo usar**: cada decisión tiene cuadro comparativo + recomendación default senior + criterios para cambiar el default + red flag.

---

## Vector DB

| Opción | Pros | Contras | Costo (orden) | Cuándo elegirla |
|---|---|---|---|---|
| **FAISS** (in-memory) | Súper rápido, simple, sin infra | Solo memoria local, no persistente nativo, sin filters complejos, sin multi-tenant | Solo compute (gratis lib) | POC, batch jobs offline, corpus chico (<1M vectores), embeddings precomputed |
| **pgvector** | Postgres = familiar, transaccional, joins, filters SQL, una sola DB | Performance cae con >10M vectores sin tuning, índices IVFFlat/HNSW menos maduros que dedicated | Costo Postgres + ~marginal | Empezás con Postgres ya, corpus <10M, querés joins relacionales, equipo no quiere otra infra |
| **Qdrant** | Self-hostable u hosted, filters nativos fuertes, payload schema, gRPC perf | Otra infra que mantener si self-hosted, comunidad menor que Pinecone | $$ self-host minimal / $$$ hosted | Multi-tenant, necesitás filters complejos, equipo OK con Rust/Docker |
| **Pinecone** | Fully managed, escala fácil, multi-region, ecosystem maduro | Vendor lock-in, costo crece rápido con escala, sin self-host | $$$$ | Equipo chico, no querés ops, escala enterprise, presupuesto OK |
| **Weaviate** | Schema rich, GraphQL, multi-modal, hybrid built-in | Más complejo conceptualmente, ops considerable si self-host | $$$ self-host / $$$$ hosted | Multi-modal (text+image), querés hybrid out of box, schema-heavy |
| **Milvus** | Escala extrema (B+ vectores), cloud-native | Complejo de operar, overkill para casos chicos | $$$$ | Corpus mil-millonarios, requisitos de latencia extremos |

**Recomendación senior (default):** **pgvector** si ya tenés Postgres y corpus <10M. **Qdrant self-hosted** si necesitás filters fuertes o multi-tenant y tu equipo aguanta ops. **Pinecone** si quieren cero ops y presupuesto cubre.

**Cuándo cambiar el default:**
- Corpus >50M y queries de latencia baja → Milvus o Qdrant tuneado
- Multi-modal nativo (text + image) → Weaviate
- POC/research → FAISS basta y es gratis

**Red flag:** elegir Pinecone "porque es lo más conocido" sin medir costo proyectado a escala — sorpresa de factura a los 6 meses.

---

## Re-ranker

| Opción | Pros | Contras | Costo (orden) | Cuándo elegirla |
|---|---|---|---|---|
| **Cohere Rerank API** | Managed, multilingual, low-effort, low latency | Vendor dependency, pricing por call | $$ (~ 0.001/call) | MVP, equipo chico, no querés self-host |
| **bge-reranker-v2-m3** | Open-source, multilingual fuerte, self-hostable | Requiere GPU para latency aceptable, ops | $-$$ (compute GPU) | Scale donde Cohere se vuelve caro, data residency, fine-tune posible |
| **Cross-encoder propio (fine-tuned)** | Best quality posible en tu dominio | Requiere data + ML expertise + maintenance | $$$ (training + serving) | Dominio muy específico (legal, médico) con eval baseline mostrando gain |
| **Skip rerank** | Cero latency adicional | Quality cae si retrieval es ruidoso | $0 | Retrieval embedding-only ya es suficiente (medido con eval), latency crítica |

**Recomendación senior (default):** **Cohere Rerank** para MVP. Migrá a **bge-reranker-v2-m3 self-hosted** cuando cost-per-call vs hosting cruce break-even.

**Cuándo cambiar el default:**
- Latencia P95 < 1s end-to-end imposible con rerank → skip + invertir en mejor retrieval
- Dominio muy específico con baselines pobres → fine-tune
- Data residency forzosa → self-host

**Red flag:** agregar re-ranker sin medir si mejora — "lo agregué porque vi un blogpost".

---

## Orchestration Framework

| Opción | Pros | Contras | Curva | Cuándo elegirla |
|---|---|---|---|---|
| **SDK directo (provider)** | Control total, sin deps, debugging directo | Reinventás patrones comunes (retry, memory, tools) | Baja | Producción crítica con un solo provider, equipo que prefiere control |
| **LangChain** | Ecosistema grande, abstractions sobre todo, prototipo rápido | Breaking changes frecuentes, ofusca a veces, performance overhead | Media | App con múltiples piezas (RAG + tools + memory), portability entre providers |
| **LangGraph** | Flujos explícitos, HITL nativo, checkpointing, time-travel | Concepto extra (state machines), overkill para agentes simples | Media-alta | Agentes complejos, multi-step, HITL, multi-agent |
| **LlamaIndex** | Retrieval-first abstractions, query engines avanzados, evaluation built-in | Menos generalista que LangChain, ecosystem orquestación menor | Media | RAG es el pilar central, indexing complejo (graph, hierarchical) |
| **Custom (sin framework)** | Tailored a tu necesidad, cero deps | Reinventás todo | Alta | Equipo expert, requisitos únicos, no encajás en frameworks |

**Recomendación senior (default):** **SDK directo + thin wrappers propios** para core. **LangGraph** para agentes complejos cuando llegan. **LlamaIndex** integrado como retriever si RAG es central.

**Cuándo cambiar el default:**
- Prototipo en horas, no semanas → LangChain
- RAG super complejo (graph, sub-question) → LlamaIndex de entrada
- Equipo grande sin AI expertise → framework opinado (LangChain) reduce decisiones

**Red flag:** adoptar LangChain "porque es popular" sin entender qué problema te resuelve — terminás peleando con la abstracción.

---

## State Management Backend

| Opción | Pros | Contras | Latencia | Cuándo elegirla |
|---|---|---|---|---|
| **In-memory (dict/var)** | Cero infra, dev velocidad | No persistente, no multi-instance, no HITL | <1ms | Dev local, tests, sesiones efímeras |
| **Redis** | Latencia bajísima, TTL nativo, pub-sub | Volátil sin persistencia config, sin transactions complejas | 1-5ms | Sesiones de chat, cache, agent state corto plazo |
| **Postgres (LangGraph checkpointer)** | Transaccional, queries, joins, durable | Latencia mayor que Redis, ops Postgres | 10-50ms | HITL, long-running, audit trail, recovery, time-travel |
| **SQLite** | Cero ops, file-based, suficiente para single instance | No multi-instance | 1-10ms | Dev / single-node prod / edge |
| **DynamoDB / Firestore** | Managed, escala automático, multi-region | NoSQL constraints, pricing complicado | 5-20ms | AWS/GCP heavy stack, no querés ops DB |

**Recomendación senior (default):** **Postgres** con LangGraph checkpointer para producción. **Redis** para session cache + Postgres como source of truth.

**Cuándo cambiar el default:**
- Latencia crítica (<10ms) → Redis only con backups periódicos a S3
- Stack 100% AWS sin Postgres → DynamoDB
- Edge deployment → SQLite

**Red flag:** state-management in-memory en producción "porque no tenemos tantos users" — el primer restart te muestra el problema.

---

## LLM Provider

| Opción | Pros | Contras | Costo input/output | Cuándo elegirla |
|---|---|---|---|---|
| **OpenAI (GPT-4o, GPT-4o-mini)** | Ecosistema mayor, function calling maduro, Structured Outputs strict | Vendor lock-in, contexto 128K (vs 200K Claude) | $$ (gpt-4o ~$2.5/M in) | Default safe, function calling-heavy, JSON strict mode |
| **Anthropic (Claude Sonnet/Opus/Haiku)** | Contexto 200K, prompt caching mejor (90% off), strong reasoning, mejor para code | Function calling sutilmente distinto, menos integraciones | $$ (sonnet ~$3/M in) | Long contexts, code-heavy, agentes con prompt caching, mejor reasoning |
| **Google Gemini (Flash/Pro)** | Contexto 1M+, multi-modal nativo, precio agresivo | API menos madura, ecosystem chico | $ (flash ~$0.075/M in) | Long context extreme, multi-modal, cost-sensitive |
| **DeepSeek (V2/V3)** | Open weights, costo bajísimo via API o self-host | Quality menor en algunos benchmarks, ecosistema chico | $ (~$0.27/M in API) o self-host | Cost-sensitive, self-host requirement, prototipos |
| **Llama (3.x / 4) self-hosted** | Open, sin vendor lock-in, data on-prem | Ops considerable, GPU costoso, quality vs frontier | Infra GPU ($$$) | Compliance/data residency forzosa, escala donde ops + GPU < API cost |
| **Cohere** | Multilingual fuerte, rerank propio, enterprise focus | Menos popular, ecosystem chico | $$ | Multilingual heavy, integración con rerank, enterprise |

**Recomendación senior (default):** **Anthropic Claude Sonnet** para agentes complejos (mejor reasoning + prompt caching). **OpenAI GPT-4o** para function-calling pesado y strict mode. **GPT-4o-mini / Haiku** para clasificación y routing.

**Cuándo cambiar el default:**
- Long context (>200K) → Gemini
- Data residency forzosa → Llama self-host
- Cost-extreme → DeepSeek o Gemini Flash
- Multi-modal central → Gemini o GPT-4o

**Red flag:** elegir "el más barato" sin medir quality en tu use case — quality pobre cuesta más en retries y user churn.

---

## Embeddings

| Opción | Pros | Contras | Dims | Costo / 1M tokens | Cuándo elegirla |
|---|---|---|---|---|---|
| **OpenAI text-embedding-3-large** | Multilingual fuerte, dimensional flexibility (Matryoshka) | Vendor, costo medio | 3072 (configurable) | ~$0.13 | Default safe |
| **OpenAI text-embedding-3-small** | Barato, multilingual decente, 80% quality de large | Dims reducidas | 1536 | ~$0.02 | Cost-sensitive, escala grande |
| **bge-large-en / bge-m3 (multilingual)** | Open-source, self-host, top en MTEB | Requiere infra | 1024 | Compute self-host | Self-host requirement, data residency |
| **Cohere embed-multilingual-v3** | Multilingual, integración con rerank Cohere | Vendor | 1024 | ~$0.10 | Multilingual, ya usás Cohere |
| **sentence-transformers (all-MiniLM)** | Tiny, super fast, gratis | Quality menor que frontier | 384 | Self-host | Edge, prototipos, search interno chico |

**Recomendación senior (default):** **text-embedding-3-small** para arrancar (cost-effective). Upgradeá a **3-large** si eval muestra gain claro. **bge-m3** si self-host requerido.

**Cuándo cambiar el default:**
- Dominio super específico con eval pobre → fine-tune un sentence-transformer
- Self-host obligatorio → bge-large / bge-m3
- Costos extremos en embedding → small + Batch API

**Red flag:** cambiar de modelo de embedding sin re-embedding del corpus completo — distancias no son comparables.

---

## Streaming Protocol

| Opción | Pros | Contras | Cuándo elegirla |
|---|---|---|---|
| **SSE (Server-Sent Events)** | HTTP/HTTPS estándar, simple, soporte browser nativo, unidireccional ideal para tokens | Solo server→client, sin binario nativo | Streaming de tokens LLM (default) |
| **WebSocket** | Bidireccional, low-overhead, binary support | Más complejo (handshake, ping/pong, reconnection), proxies a veces hostiles | Interrupciones del usuario mid-stream, multi-user collab, voice/video |
| **Polling** | Trivial, funciona en cualquier infra | Latencia mala, costo de requests | Fallback only cuando SSE/WS no funcionan |
| **gRPC streaming** | Binary, bidir, schema-based | Cliente JS complejo, no nativo browser | Servicio interno (no browser) |

**Recomendación senior (default):** **SSE** para chat token streaming. **WebSocket** solo cuando hay interrupciones del usuario en mitad de la respuesta.

**Cuándo cambiar el default:**
- Necesitás interrumpir mid-stream desde cliente → WebSocket
- Servicio interno backend-to-backend con streaming → gRPC

**Red flag:** WebSocket "por defecto" sin necesidad bidir — agregás complejidad de reconnection, heartbeats, scaling.

---

## Cost Optimization Strategy

| Opción | Cuándo aplicar | Ahorro típico | Esfuerzo |
|---|---|---|---|
| **Prompt caching (Anthropic)** | System prompt o context largo y estable | 60-90% input cost | Bajo |
| **Batch API (OpenAI/Anthropic)** | Workloads non-realtime (24h SLA) | 50% | Bajo |
| **Model routing (chico → grande)** | Mix de queries fáciles + difíciles | 30-70% global | Medio |
| **Context compression (LongLLMLingua)** | Contextos enormes en cada call | 20-40% input cost | Medio |
| **Response caching (semantic)** | Queries repetidas frecuentes | 30-50% en queries cacheables | Medio |
| **Smaller embeddings** | Escala embedding-heavy | 6x con 3-small vs 3-large | Bajo (testear quality) |
| **Self-host LLM** | Escala extrema donde API > GPU | Variable | Alto (ops) |

**Recomendación senior (default):** orden de prioridad — (1) **prompt caching**, (2) **batch API** donde aplique, (3) **model routing** con classifier barato. Resto solo si los anteriores no alcanzan.

**Cuándo cambiar el default:**
- Cost ya bajo y feature delivery prioritario → no optimices prematuramente
- Compliance fuerza self-host → skip lo demás, focus en infra

**Red flag:** self-hosting LLM "para ahorrar" sin contar costo de ops, GPU underutilization, expertise requerida.

---

## Multi-Agent Topology

| Topología | Pros | Contras | Cuándo elegirla |
|---|---|---|---|
| **Single agent + many tools** | Simple, debugging directo, latencia baja | Limita a ~10 tools antes de confundir routing | Apps simples, tools homogéneas, MVP |
| **Supervisor + workers (estrella)** | Modular, workers especializados, escalable | Supervisor puede ser bottleneck o god-object | 3-10 workers con dominios distintos, app moderada |
| **Hierarchical (supervisores anidados)** | Maneja muchos workers, domain separation | Latencia (más hops), debugging complejo, costo | >15 workers, multi-dominio claro |
| **Horizontal network (P2P/blackboard)** | Emergent, robusto a falla individual, no single point | Impredecible, convergencia incierta, hard to test | Simulación, debate, research, no SLA crítico |

**Recomendación senior (default):** empezá con **single-agent + few tools**. Subí a **supervisor** cuando excedés ~10 tools o necesitás separación clara. **Hierarchical** solo cuando supervisor único no llega. **Horizontal** rara vez en producción.

**Cuándo cambiar el default:**
- Domain clearly separated (ej. marketing vs eng vs analytics) → supervisor de entrada
- Research / exploration → horizontal
- SLA strict y predictability crítica → single-agent o supervisor (nunca horizontal)

**Red flag:** "vamos a hacer multi-agent" sin justificar por qué un single-agent con tools no alcanza — complejidad gratis.

---

## Eval Framework

| Opción | Pros | Contras | Pricing | Cuándo elegirla |
|---|---|---|---|---|
| **LangSmith (LangChain)** | Tracing + evals + datasets integrados, UI buena | Vendor, paid tier para serio uso | Free tier limitado, paid $$$ | Stack LangChain, equipo OK con managed |
| **Langfuse** | Open-source, self-hostable, tracing + evals + prompt management | Self-host = ops | Free self-host / paid cloud | Self-host requirement, open ethos, multi-framework |
| **Promptfoo** | CLI-first, declarativo (YAML), CI-friendly, gratis | Menos integrado con tracing | Free OSS | CI evals, regression testing, sin tracing pesado |
| **Braintrust** | Modern UX, fast, dataset management strong | Vendor, paid | Paid $$$ | Equipo data-science-y, dataset-heavy |
| **Custom (scripts propios)** | Tailored, cero deps | Reinventás, sin UI | Free | Casos específicos, equipo expert |

**Recomendación senior (default):** **Langfuse self-hosted** (open, integra todo, no vendor lock-in). **Promptfoo** para CI gates. **LangSmith** si ya estás all-in LangChain y no querés ops.

**Cuándo cambiar el default:**
- Equipo no quiere ops → LangSmith o Langfuse cloud
- Eval simple en CI → Promptfoo basta
- Data scientist team con dataset focus → Braintrust

**Red flag:** producción sin eval framework, "después lo agregamos" — debt acumulada que no se paga.

---

## Memory Backend

| Opción | Pros | Contras | Cuándo elegirla |
|---|---|---|---|
| **Vector-based (embeddings + similarity)** | Retrieval por semantic relevance, escala bien | Sin estructura relacional, distancia ≠ relevancia siempre | Episodic memory genérico, "remember conversations like this" |
| **Graph-based (Neo4j, knowledge graph)** | Capturas relaciones (X causa Y, A trabaja con B), traversal | Construcción manual o LLM-extracted (errores), queries complejas | Semantic memory rich, relaciones importan (CRM, knowledge bases) |
| **Hybrid (vector + graph + SQL)** | Best of all, queries por similarity + structure | Más infra, más complejidad | Production-grade memory para agentes pesados |
| **SQL-only** | Familiar, transactional, queries declarativas | Sin semantic, dependés de keywords | Memory estructurada simple, user profiles |
| **Key-value (Redis)** | Rápido, simple, TTL | Sin queries complejas | Working memory de turno actual, scratchpads |

**Recomendación senior (default):** **vector-based** para episodic. **SQL** para semantic estructurado (user profile, preferences). **Redis** para working. Híbrido se construye combinando.

**Cuándo cambiar el default:**
- Relaciones complejas con traversal (causal chains, org charts) → graph
- Memory es central al producto → hybrid investment vale

**Red flag:** todo en vector DB sin pensar estructura — semantic memory que debería ser SQL queries terminan siendo búsquedas de embedding ruidosas.

---

## Function Calling vs Structured Outputs

| Opción | Para qué sirve | Costo | Cuándo elegirla |
|---|---|---|---|
| **Function calling / Tool use** | Agent loop, LLM decide qué tool llamar, posibles iteraciones | Multiple round-trips si hay ReAct | Necesitás acciones (read DB, call API, execute), iteración |
| **Structured Outputs (strict JSON schema)** | Una respuesta tipada en formato exacto | Una sola call | Extracción de datos, clasificación, transformación |
| **JSON mode básico** | JSON válido pero schema no garantizado | Una call + parser tolerante | Cuando strict no soportado o sobra |
| **Prompt-based JSON** | "Devuelve JSON: ..." sin guarantees | Una call + parser frágil | Solo si nada mejor disponible |

**Recomendación senior (default):** **Structured Outputs strict** para data extraction. **Tool use** para agentes con acciones. Combiná: tool use cuyo tool retorna structured data.

**Cuándo cambiar el default:**
- Provider no soporta strict → JSON mode + validation post-hoc
- Schema ultra-dinámico (cambia per request) → prompt-based con validation aggressive

**Red flag:** parsear LLM text con regex "porque structured outputs no me dejaba algo" — casi siempre podés modelar el schema mejor.
