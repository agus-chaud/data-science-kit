# Hito 4 — Orquestación

## Por qué importa (perspectiva corporativa)

Mirá, te voy a ser brutalmente honesto: este hito es el que decide si sos un AI Engineer **Mid** o un AI Engineer **Senior**. Mid sabe armar un agente con LangChain copiando del cookbook. Senior sabe cuándo NO usar LangChain, cuándo migrar a LangGraph, cuándo escribir su propia orquestación en 200 líneas porque el framework agrega más fricción que valor. Y sabe ARQUITECTURALMENTE por qué.

En empresas reales esto se traduce en: el equipo arrancó con LangChain porque "es lo que se usa", a los 6 meses tienen 3000 líneas de Chains imposibles de debuggear, los agentes fallan en producción y nadie sabe en qué paso del workflow se rompió. La empresa contrata a un consultor senior (vos, si dominás este hito) que migra a LangGraph con state explícito + checkpointing y de repente pueden inspeccionar cada step, hacer human-in-the-loop, retomar después de un crash. Eso es el delta que pagan plata por.

LangGraph en particular es **EL framework de orquestación serio de 2025-2026**. Anthropic, LangChain Inc, y la mayoría de productos LLM serios (Replit, Klarna's customer service, etc) lo usan. LlamaIndex tiene su lugar en RAG-heavy apps. CrewAI, AutoGen, AutoGPT son más juguetes-de-investigación que herramientas serias para producción. Saber esta jerarquía y defenderla con argumentos técnicos te diferencia. Oportunidades laborales: "Senior AI Engineer" en cualquier empresa con agentes multi-step en producción, "Solutions Architect" para vendors (LangChain Inc directamente contrata), consultor de "rescate de proyectos LangChain mal hechos" — esto último PAGA porque casi todos los proyectos pioneros están atascados.

## Conceptos de este hito

### langchain-basics

**Qué es**: Framework de Python/JS para componer chains de LLM calls con un DSL declarativo (**LCEL** — LangChain Expression Language). Provee Runnables, abstracciones de prompts, output parsers, integraciones a 100+ vendors.

**La trampa del junior**: Usar LangChain para TODO incluso cuando es overkill. "Llamar a OpenAI con un prompt simple" no necesita LangChain — necesita `openai.chat.completions.create()`. LangChain agrega 3 capas de abstracción que vuelven el código menos legible para tareas simples.

**Cómo lo piensa un senior**: LangChain tiene **valor real** cuando: (1) componés ≥3 LLM calls con lógica entre ellas, (2) integrás múltiples vendors y querés swap fácil (`ChatOpenAI()` → `ChatAnthropic()` cambia 1 línea), (3) usás muchos retrievers/loaders pre-construidos (PDF, Notion, Slack, etc), (4) querés tracing automático con LangSmith. NO tiene valor cuando: (a) hacés 1-2 calls simples, (b) querés control total del prompt sin que el framework te meta texto extra, (c) tu equipo no se va a tomar el tiempo de aprender LCEL. Junior elige por defecto. Senior elige por contexto.

**Tradeoffs reales**:

| Framework | Cuándo brilla | Cuándo es overkill |
|---|---|---|
| LangChain (LCEL) | Composición de chains, multi-vendor swap, retrievers built-in | 1-2 LLM calls simples |
| LangGraph | Workflows con branching, HITL, state persistente | Workflows lineales puros |
| LlamaIndex | RAG-heavy apps, parsers de docs sofisticados | Apps no-RAG-centradas |
| Pydantic AI | Type safety estricta, agents typed end-to-end | Multi-modal, complex workflows |
| Sin framework (raw SDK) | Casos simples, control total, MVP rápido | Composición compleja repetida |
| LiteLLM | Solo querés abstraer providers, no chains | Necesitás composición |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es LCEL y por qué usar `|` para componer?*
  A: LangChain Expression Language. El `|` (pipe) compone Runnables en una pipeline: `prompt | llm | parser`. Es declarativo: cada paso es un Runnable con `.invoke()`, `.stream()`, `.batch()`, `.ainvoke()` uniformes. Beneficio: streaming, async, batch funcionan "gratis" en cualquier composición; tracing automático en LangSmith.
- Q (senior): *¿Cuándo NO usás LangChain?*
  A: (1) Cuando la lógica es 1-2 LLM calls sin composición real — agrega abstracción sin valor. (2) Cuando necesito control milimétrico del prompt (LangChain a veces inyecta texto en system prompts de chains). (3) Cuando el equipo no va a invertir tiempo en aprender LCEL y va a terminar peleándose con el framework. (4) Cuando la curva de upgrade entre versiones es problema (LC tiene historial de breaking changes entre 0.0.x → 0.1 → 0.2 → 0.3).
- Q (trampa): *¿LangChain vs OpenAI SDK directo?*
  A: Trampa de falsa dicotomía. Coexisten. SDK directo para llamadas simples, LangChain para composición que se reutiliza. Y en LangChain podés usar el cliente SDK directo cuando necesitás features que el wrapper no expone. La pregunta correcta es "¿cuál es la unidad de composición acá?".

### langgraph-dags

**Qué es**: Framework para definir agentes como **grafos de estado explícitos**. Nodos = funciones que mutan estado. Edges = transiciones (condicionales o fijas). Estado = TypedDict o Pydantic compartido. Compila a un StateGraph ejecutable con checkpointing, HITL, time travel.

**La trampa del junior**: Hacer agentes con loops `while True` ad-hoc y arrays de mensajes que crecen sin control. Funciona para demo, imposible de mantener en producción. O al revés: usar LangGraph para workflows triviales (3 pasos lineales) donde una función Python alcanzaba.

**Cómo lo piensa un senior**: LangGraph es **state machine explícita para agents**. El valor central es **observabilidad y control**: vos ves exactamente qué nodo se ejecutó, en qué orden, qué cambió en el state. Permite: branching condicional (decide qué nodo sigue según state), parallel branches (varios nodos en paralelo mergeando state), checkpointing (persiste state entre turnos / sobrevive crashes), human-in-the-loop (pausa en un nodo, espera input humano, sigue), time travel debugging (volvés a un step pasado y modificás). Es **infra de production** para agents serios.

**Tradeoffs reales**:

| Approach | Cuándo |
|---|---|
| LangGraph DAG | Agents con branching, HITL, retries, state persistente |
| AgentExecutor de LangChain (legacy) | Casos simples ReAct, deprecated en favor de LangGraph |
| Custom state machine | Control total, dependencias mínimas, tooling propio |
| CrewAI / AutoGen | Prototipos rápidos multi-agent, no production |
| Workflows simples sin framework | 3-5 pasos lineales, no necesitás checkpointing |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué LangGraph y no un loop ReAct simple?*
  A: Porque hace explícitos el state, las transiciones y el control flow. En un loop ReAct ad-hoc, todo está implícito en el código — para debug tenés que leer el código. En LangGraph el grafo se compila y podés visualizarlo (`graph.get_graph().draw_mermaid()`), trazarlo en LangSmith, hacer checkpoints, retomar.
- Q (senior): *¿Cómo modelás el state de un agente complejo en LangGraph?*
  A: Empiezo con un TypedDict mínimo (messages, context_for_this_turn). Voy agregando claves a medida que el grafo lo pide. Si el state crece, lo separo en sub-states por área (input, intermediate, output) y uso reducers (`Annotated[list, add_messages]`) para mergear updates de múltiples nodos. Cuidado con state que crece sin bound — implementar summarization o ttl para campos efímeros.
- Q (trampa, system design): *Un workflow con 8 nodos secuenciales. ¿LangGraph o función Python con 8 awaits?*
  A: Si es PURAMENTE secuencial sin branching, sin HITL, sin necesidad de retomar — función Python alcanza. LangGraph agrega valor cuando hay (a) branching condicional, (b) parallel nodes con merge, (c) HITL, (d) checkpointing necesario, (e) querés observabilidad granular en LangSmith. Si nada de esto aplica, LangGraph es ceremonia. La trampa: usar LangGraph "porque queda bien en el CV".

### state-management

**Qué es**: Cómo modelás, propagás y mutás el estado a través de los nodos del agente. En LangGraph: TypedDict + reducers para merge concurrente. En sistemas custom: dataclass / Pydantic + funciones puras (`new_state = update(state, event)`).

**La trampa del junior**: State como diccionario libre que cada nodo modifica in-place. A los 5 nodos nadie sabe qué tiene `state` realmente. Tipos perdidos, race conditions en branches paralelos, bugs imposibles de reproducir.

**Cómo lo piensa un senior**: State es **el contrato entre nodos**. Senior aplica: (1) **schema explícito** (TypedDict / Pydantic) — sabés qué hay y de qué tipo, (2) **reducers para concurrencia** — si dos nodos paralelos escriben al mismo campo, definís cómo se mergea (concatenar listas, last-write-wins, max, custom merge), (3) **immutability** conceptual — los nodos devuelven updates, no mutan in-place, (4) **bounded growth** — campos que crecen tienen política (sliding window, summarization), (5) **separation of concerns** — state ≠ memory ≠ context window. Cada uno es cosa distinta.

**Tradeoffs reales**:

| Pattern | Cuándo |
|---|---|
| TypedDict + reducers (LangGraph) | Default para LangGraph workflows |
| Pydantic BaseModel | Validación estricta + serialización |
| Immutable dataclass + pure functions | Control total, fácil testear |
| Dict libre | Solo prototipos, deuda técnica garantizada |
| Mensaje pasing (Actor model) | Multi-agent peer-to-peer |
| External state store (Redis) | State compartido entre múltiples agent instances |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es un reducer en LangGraph state?*
  A: Función que define cómo se mergean updates concurrentes a un campo. Ej `Annotated[list[Message], add_messages]` — si dos nodos paralelos agregan mensajes, se concatenan correctamente en vez de pisarse. Sin reducer, el comportamiento default es last-write-wins, que en parallel branches lleva a perder datos.
- Q (senior): *Tu state tiene un campo `intermediate_results` que crece a 50k tokens en agentes largos. ¿Qué hacés?*
  A: Tres approaches: (1) **summarization periódica**: cada N steps, un nodo "compactor" resume el campo y reemplaza. (2) **sliding window**: mantener solo últimas K entradas, las viejas a episodic memory externa. (3) **separar** state "vivo" (current turn) del state "histórico" (a Redis/Postgres). La elección depende de si el LLM necesita ver los intermedios o solo el resumen.
- Q (trampa): *Diferencia entre state, memory y context window.*
  A: **State** = datos del grafo en ejecución, controlado por tu app. **Memory** = persistencia entre conversaciones (working/episodic/semantic — ver Hito 1). **Context window** = el slice que efectivamente metés en el prompt del LLM. Los tres son cosas distintas y pueden tener tamaños y stores distintos. Confundirlos es un smell de diseño pobre.

### checkpointing

**Qué es**: Persistir el state del agente en cada step (o en steps marcados) para: (a) **sobrevivir crashes** (retomar donde quedó), (b) **human-in-the-loop** (pausar en un nodo, esperar approval humano, seguir), (c) **time travel debugging** (volver a un step anterior, mutarlo, re-ejecutar), (d) **conversaciones persistentes** (entre turnos del usuario, en chats).

**La trampa del junior**: No persistir nada y guardar todo en memoria del proceso. Crash del worker = se pierde todo. O hacer su propio sistema de persistencia con pickle y archivos sueltos. Inmantenible.

**Cómo lo piensa un senior**: Checkpointing es **non-negotiable en agents production**. Backends: LangGraph tiene `MemorySaver` (dev), `SqliteSaver` (single-machine), `PostgresSaver` (production multi-instance), `RedisSaver`. El **thread_id** identifica una conversación/workflow específico — usás `user_id` o `session_id` típicamente. Permite: resume mid-workflow, multi-turn chat con state continuo, HITL con timeouts de horas/días, debugging post-mortem reproduciendo el state exacto.

**Tradeoffs reales**:

| Checkpointer | Cuándo |
|---|---|
| MemorySaver | Dev/test only — se pierde al reiniciar |
| SqliteSaver | Single-machine apps, prototipos serios |
| PostgresSaver | Producción multi-instance, transactions |
| RedisSaver | Alta concurrencia, TTL nativo |
| Custom (S3, etc) | Casos exóticos, compliance específica |

**En entrevista te van a preguntar**:
- Q (mid): *¿Para qué sirve un checkpointer en LangGraph?*
  A: Persistir state entre steps del grafo, lo que permite (a) retomar después de un crash o pausa, (b) implementar HITL (parar en un nodo, esperar input humano, seguir), (c) tener conversaciones persistentes entre turnos del usuario, (d) time-travel debugging.
- Q (senior): *Implementás HITL en un workflow de aprobación de gastos. ¿Cómo lo armás con LangGraph?*
  A: (1) Nodo "draft_decision" propone aprobar/rechazar y guarda en state. (2) `interrupt_before=["execute"]` antes del nodo que ejecuta. (3) Checkpoint con PostgresSaver + thread_id = expense_id. (4) Frontend muestra la decisión propuesta + botón approve/reject del humano. (5) Cuando humano decide, app llama `graph.update_state(thread_id, {"decision": human_choice})` + `graph.invoke(None, config)` para resumir. (6) Nodo "execute" lee el state actualizado y ejecuta. Cubre crashes, pausa indefinida, multi-user.
- Q (trampa, system design): *¿Cuándo el costo de checkpointing supera al beneficio?*
  A: Casos: (1) workflows muy cortos (<1s end-to-end) donde el overhead de write a Postgres > tiempo de ejecución, (2) state muy grande (MB) escrito en cada step → I/O brutal, (3) workflows fire-and-forget donde recovery no importa. Solución: checkpointear solo en steps "interesantes" (no en todos), o usar MemorySaver para sub-grafos efímeros y persistente solo para el top-level.

### llamaindex-vs-langchain

**Qué es**: Las dos librerías mainstream del ecosistema LLM Python. **LangChain** = orquestación general, multi-vendor, chains/agents. **LlamaIndex** = retrieval-first, parsers de docs sofisticados, indexing strategies. No son competencia directa — son complementarias pero hay overlap.

**La trampa del junior**: Elegir uno por "popularidad" o porque "lo vi en un tutorial". Después se da cuenta que para su use case el otro es 3x menos código.

**Cómo lo piensa un senior**: Decisión basada en **dónde está el peso del problema**. Si el problema es 80% RAG (chunking complejo, multi-modal, query engine sofisticado, response synthesis) → LlamaIndex tiene las abstracciones mejores. Si el problema es 80% orquestación (chains, agents, multi-vendor) → LangChain/LangGraph. Si es 50/50 → podés combinar (LlamaIndex como retriever DENTRO de un agente LangChain). En 2025-2026 muchas empresas hacen exactamente eso: LlamaIndex para el RAG layer, LangGraph para el orchestration layer.

**Tradeoffs reales**:

| Librería | Mejor en | Más débil en |
|---|---|---|
| LangChain | Multi-vendor abstraction, chains, integraciones (300+) | RAG sofisticado (chunking, query engines) |
| LangGraph | Workflows con state, HITL, checkpointing | Curva de aprendizaje, ceremonia para casos simples |
| LlamaIndex | Document parsing (LlamaParse), query engines, advanced retrieval | Orquestación de agents multi-step compleja |
| Pydantic AI | Type safety, agents typed | Ecosistema joven, menos integraciones |
| Haystack | Pipelines de búsqueda + LLM, on-prem fuerte | Menos popular en startups |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué diferencia hay entre LangChain y LlamaIndex?*
  A: LangChain es orquestación general (chains, agents, multi-vendor abstraction). LlamaIndex es retrieval-first (chunking, indexing, query engines, doc parsing). No son rivales — se combinan. Para RAG complejo, LlamaIndex tiene mejores abstracciones (SubQuestionQueryEngine, RecursiveRetriever). Para orquestación de agents, LangGraph.
- Q (senior): *Tenés un proyecto: chatbot interno con RAG sobre 10k PDFs legales + 5 tools (DB queries, API calls). ¿Qué stack elegís?*
  A: LlamaIndex para el RAG layer: LlamaParse para PDFs (mejor que pypdf para tablas/multi-column), nodes con hierarchical chunking, hybrid retriever (BM25 + dense), re-ranker. LangGraph para el agent layer: nodos por tool, state explícito, checkpointing en Postgres para multi-turn. El RAG aparece como una tool ("knowledge_search") dentro del agente. Justificación: cada framework hace lo que hace mejor.
- Q (trampa): *¿Y CrewAI o AutoGen?*
  A: Trampa para ver si caés en hype. Mi respuesta: para producción seria, evito ambos. CrewAI y AutoGen son buenos para prototipos rápidos multi-agent y research, pero les falta madurez en observabilidad, error handling, y deployment patterns. LangGraph con multi-agent (Hito 5) cubre los mismos casos con mejor control. Si me obligan, AutoGen tiene research backing (Microsoft) más sólido que CrewAI.

## Lo que el libro hace bien acá

- **chapter02** — `The Agent Engineer's Toolkit` — comparison matrix de LangChain / LangGraph / LlamaIndex / AutoGPT / CrewAI / AutoGen. Buena introducción al ecosistema. NO entra a profundidad en LangGraph DAGs ni checkpointing.
- **chapter07** — `Tool Manipulation & Orchestration Agents` — implementa Tool-Using Agent, Chain-of-Agents, Agentic Workflows con state sharing y HITL gates. Es lo más cercano a state management explícito en el libro. Útil para ver el concepto, aunque el código está en raw Python no en LangGraph.
- **chapter15** — `Education & Knowledge Agents` — POMDP tutor con Bayesian Knowledge Tracing es un ejemplo de state management complejo (no LLM-native, pero el patrón aplica). El collective intelligence muestra coordinación multi-step.

## Lo que el libro NO tiene (gaps a saber)

- **LangChain LCEL profundo**: el libro lo menciona, no enseña.
  - Recurso: https://python.langchain.com/docs/concepts/lcel/
  - Qué entender: Runnable protocol, composición con `|`, RunnableParallel, RunnableBranch, streaming y batch automáticos, cómo debuggear chains con LangSmith.

- **LangGraph DAGs**: gap CRÍTICO. El libro implementa workflows en raw Python.
  - Recurso: https://langchain-ai.github.io/langgraph/tutorials/introduction/ + https://langchain-ai.github.io/langgraph/concepts/low_level/
  - Qué entender: StateGraph, nodes vs edges (conditional edges), reducers en state, compile + invoke + stream, visualización Mermaid del grafo.

- **Checkpointing y HITL**: el libro lo menciona en chapter07 pero no implementa serio.
  - Recurso: https://langchain-ai.github.io/langgraph/concepts/persistence/ + https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/breakpoints/
  - Qué entender: thread_id, checkpointer backends (Sqlite/Postgres/Redis), `interrupt_before` / `interrupt_after`, `graph.update_state`, time travel debugging con `graph.get_state_history`.

- **LlamaIndex query engines avanzados**: ausente.
  - Recurso: https://docs.llamaindex.ai/en/stable/module_guides/deploying/query_engine/
  - Qué entender: SubQuestionQueryEngine (descompone queries complejas), RouterQueryEngine (rutea por tipo), RecursiveRetriever (multi-hop), Citation engines.

- **Comparación arquitectónica de frameworks**: el libro lista, no compara con criterio.
  - Recurso: https://blog.langchain.dev/langgraph-multi-agent-workflows/ + posts comparativos como Pydantic AI vs LangChain (Latent Space podcast cubre).
  - Qué entender: cuándo cada framework gana, ejemplos concretos de empresas, anti-patterns por framework.

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `langchain-basics`: NO hay notebook directo del libro en LangChain pura. Tomá un agente del chapter01 o chapter05 y reescribilo usando LCEL básico (`prompt | llm | parser`). Pegame ambas versiones y explicame qué se hizo más simple y qué se hizo más complicado.
- `langgraph-dags`: hacé el quickstart oficial de LangGraph (https://langchain-ai.github.io/langgraph/tutorials/introduction/). Implementá un grafo de 3 nodos con UN conditional edge. Visualizalo (`graph.get_graph().draw_mermaid()`). Pegame el grafo + un trace de ejecución.
- `state-management`: en el grafo del punto anterior, agregá un campo `messages: Annotated[list, add_messages]` y otro campo `iterations: int` con reducer `operator.add`. Forzá una corrida con dos nodos paralelos escribiendo a ambos campos. Mostrame que se mergea correcto.
- `checkpointing`: agregá `SqliteSaver` al grafo. Invocá con `thread_id="test1"`, matá el proceso a mitad, levantalo, retomá con el mismo thread_id. Mostrame el state recuperado.
- `llamaindex-vs-langchain`: implementá EL MISMO RAG simple (5 docs, 3 queries) con LlamaIndex y con LangChain. Compará líneas de código + UX del API. Pegame ambas implementaciones + tu opinión.

### Para subir a `mastered`

- `langchain-basics`: en un proyecto real, justificá la elección de LangChain (o NO usarlo) con análisis de la cantidad de chains, multi-vendor needs, y curva del equipo. Feynman check: explicáme en 2 minutos qué es LCEL a alguien que no sabe LangChain.
- `langgraph-dags`: implementá un agente real con ≥5 nodos, ≥2 conditional edges, ≥1 parallel branch. Defendé cada decisión de modelado del grafo. Documentá el state schema completo.
- `state-management`: en proyecto propio, hacé refactor de state libre (dict) a TypedDict con reducers. Mostrame el antes/después y los bugs que cazó el refactor.
- `checkpointing`: implementá HITL real en un workflow (no demo) con PostgresSaver + frontend que muestra el state y permite approve/reject + resume desde el state actualizado. Documentá el threat model (qué pasa si dos humans intervienen concurrente, etc).
- `llamaindex-vs-langchain`: en proyecto real, defendé tu stack final con análisis de tradeoffs documentado. Si combinaste (LlamaIndex retriever dentro de LangGraph agent), explicáme por qué a nivel de arquitectura.

## Anti-patterns que vas a ver en clientes reales

1. **LangChain para llamar 1 vez a OpenAI**
   - Cómo se hace: `from langchain.chat_models import ChatOpenAI; llm = ChatOpenAI(); llm.invoke(prompt)` cuando alcanzaba `openai.chat.completions.create(...)`.
   - Por qué se hace: "es lo que se usa para LLMs en Python".
   - Costo real: dependency tree gigante (LangChain + 50 transitive), tiempo de import +2s, abstracciones que ocultan el prompt real que se manda.
   - Cómo lo arregla un senior: usar SDK directo para casos simples, LangChain solo cuando hay composición real (≥3 calls con lógica).

2. **AgentExecutor de LangChain con prompts custom para workflows complejos**
   - Cómo se hace: armar agentes con `initialize_agent(agent_type="zero-shot-react-description")` y muchas tools custom.
   - Por qué se hace: legacy approach pre-LangGraph.
   - Costo real: imposible debuggear cuando falla, no hay state explícito, no hay HITL, no hay checkpointing. Cuando el agente entra en loop, no sabés por qué.
   - Cómo lo arregla un senior: migrar a LangGraph. AgentExecutor está deprecated by LangChain Inc — ellos mismos recomiendan LangGraph para todo lo serio.

3. **State libre como diccionario sin schema**
   - Cómo se hace: `state = {}` y cada nodo hace `state["loquefuere"] = ...`.
   - Por qué se hace: "es Python, total".
   - Costo real: a los 8 nodos nadie sabe qué claves hay; cambios de un nodo rompen otros silenciosamente; race conditions en parallel branches imposibles de reproducir.
   - Cómo lo arregla un senior: TypedDict o Pydantic con todos los campos explícitos y reducers donde hay paralelismo.

4. **Cero checkpointing, todo en memoria**
   - Cómo se hace: agente corre como script Python sin persistencia.
   - Por qué se hace: "es solo un agent batch".
   - Costo real: crash del worker = se pierde el trabajo de horas. HITL imposible. Multi-turn chat no escala (state per request en memoria → no funciona detrás de load balancer con N réplicas).
   - Cómo lo arregla un senior: PostgresSaver o RedisSaver desde día 1 en producción. Thread_id por conversación. El overhead es mínimo y abre opciones críticas (resume, HITL, debug).

5. **Mezclar LangChain + LlamaIndex sin separación clara de capas**
   - Cómo se hace: usar LlamaIndex para retrieval pero embeber su agent runtime ADEMÁS de LangChain agent. Dos runtimes peleando.
   - Por qué se hace: copy-paste de tutoriales mezclados.
   - Costo real: state distribuido en dos sistemas, debugging es pesadilla, integraciones de tracing inconsistentes.
   - Cómo lo arregla un senior: separación clara — LlamaIndex SOLO como retriever (no su agent loop), LangGraph SOLO como orquestador. Llama Index expone un retriever que se invoca como tool desde un nodo LangGraph.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés decidir con criterio cuándo usar LangChain, cuándo LangGraph, cuándo LlamaIndex, cuándo combinar, y cuándo nada de eso?
- [ ] ¿Podés implementar un agente real con LangGraph que tenga state explícito, conditional edges, y checkpointing?
- [ ] ¿Sabés diseñar el state schema de un agente complejo con reducers correctos para parallel branches?
- [ ] ¿Podés implementar HITL con checkpointing + interrupt + resume usando un backend persistente?
- [ ] En entrevista senior, ¿podés defender tu elección de stack de orquestación contra preguntas de "¿y por qué no CrewAI/AutoGen/raw Python?" con argumentos técnicos sólidos?
- [ ] ¿Podés migrar un proyecto LangChain AgentExecutor legacy a LangGraph y explicar qué ganás en cada eje?
