# Hito 5 — Multi-Agent

## Por qué importa (perspectiva corporativa)

Acá hay un tema con el que tenés que tener mucho cuidado: multi-agent es **lo más overhyped** de la industria AI Engineering ahora mismo. Todo el mundo quiere "agentic swarms", "auto-organizing teams of agents", y blablablá. La realidad: el 80% de los casos donde se usa multi-agent, **un solo agent bien diseñado con tools resolvía mejor**. Multi-agent agrega latencia, costo, complejidad de debugging y modos de falla nuevos (deadlocks, infinite delegation, conflictos).

PERO — y esto es importante — hay un 20% de casos donde multi-agent es la respuesta correcta y no hay otra. Workflows con sub-tasks muy heterogéneas (research + writing + fact-checking), sistemas que requieren especialización profunda (security expert + business logic expert + ux expert revisando un código), simulaciones (multi-agent negotiation, agent-based modeling). Saber **distinguir** los dos casos es lo que separa al AI Engineer Senior del que solo replica el último blogpost de Anthropic.

En empresas reales esto se ve así: startup arma "multi-agent system" con 7 agentes especializados, demo brutal en YC, después en producción descubren que tarda 60 segundos por query, cuesta USD 0.50 por respuesta, y un solo agente con 5 tools entrega lo mismo en 8s y USD 0.05. El consultor senior que llega y reduce de 7 agentes a 1 con tools es héroe. El que llega y agrega "el 8vo agente para coordinar mejor" es despedido. Patterns clave que tenés que dominar: **supervisor**, **hierarchical**, **horizontal network**, y la decisión de **cuándo NO usar multi-agent**. Oportunidades laborales: "AI Architect" en empresas con casos genuinamente multi-agent (Anthropic Solutions, simulaciones financieras, research automation), AutoGen/CrewAI/LangGraph contributors, y posiciones senior donde tu valor es **decir que NO al multi-agent** cuando no aplica.

## Conceptos de este hito

### supervisor-pattern

**Qué es**: Un agente **"jefe"** (supervisor) recibe el input del usuario, decide qué agente especialista (worker) debe manejarlo, le delega, recibe el resultado, y decide siguiente paso o termina. Topology centralizada: todos los workers reportan al supervisor.

**La trampa del junior**: Implementar el supervisor con un prompt tipo "elegí entre estos 8 agentes" y esperar que el LLM rutee bien con zero-shot. Performance horrible: confunde agentes con descripciones parecidas, no maneja casos ambiguos, no aprende de errores pasados.

**Cómo lo piensa un senior**: Supervisor pattern es **el default razonable** para multi-agent. Diseño correcto requiere: (1) **descripciones de workers claras y MUTUAMENTE EXCLUYENTES** (si dos workers podrían atender la misma tarea, el routing va a ser inestable), (2) **prompt del supervisor con criterios explícitos** y few-shot examples de casos difíciles, (3) **fallback explícito** si no matchea ningún worker (NO inventar uno, devolver "no puedo atender esto"), (4) **routing decision logged** para análisis y debugging, (5) **limit en número de delegations** por turno (evita loops), (6) **option de classification model dedicado** (BERT chico fine-tuneado) en vez del LLM para routing si volumen es alto.

**Tradeoffs reales**:

| Approach | Cuándo |
|---|---|
| LLM-as-supervisor (prompt) | Pocos workers, baja volumen, casos heterogéneos |
| Classifier model + supervisor LLM | Alto volumen, latencia importa, workers >5 |
| Rule-based routing | Casos donde las reglas son determinísticas (intent classification con keywords) |
| Embedding-based routing | Workers descritos como embeddings, match por similarity |
| LangGraph supervisor template | Production-ready, observability, checkpointing |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es el supervisor pattern?*
  A: Topología multi-agent donde un agente central (supervisor) recibe queries y delega a agentes especializados (workers) según el tipo de tarea. Después recolecta el resultado y decide si terminar o delegar de nuevo. Es la topología más simple y más usada en producción.
- Q (senior): *¿Cómo evitás que el supervisor entre en loop delegando entre 2 workers?*
  A: (1) Max delegations per turn (hard limit). (2) En el state del grafo, trackear historial de delegations y mostrárselo al supervisor en el prompt — "ya delegaste a worker_A 3 veces sin resultado, considerá terminar o escalar". (3) Si dos workers se mandan trabajo entre sí, eso es smell de bad design — sus responsabilidades no son disjuntas, hay que refactorear. (4) Implementar circuit breaker que mata el workflow si supera threshold y devuelve error contextualizado.
- Q (trampa, system design): *¿Supervisor LLM o classifier dedicado para routing?*
  A: Depende del volumen y latencia. Bajo volumen / casos muy variados → LLM-as-supervisor (flexibilidad gana). Alto volumen (>10 RPS) / casos relativamente estables → classifier dedicado (BERT fine-tuneado) baja latencia 10x y costo 100x. Patrón híbrido: classifier rutea, supervisor LLM solo interviene cuando la confidence del classifier es <threshold. La trampa: usar LLM "porque es lo cool" cuando un classifier de 50ms resolvía.

### hierarchical-pattern

**Qué es**: Supervisores **anidados**. Un top-supervisor delega a sub-supervisors que a su vez tienen sus propios workers. Jerarquía de N niveles. Cada nivel maneja su propio scope.

**La trampa del junior**: Hacer jerarquía profunda "por si acaso". 4 niveles de supervisors anidados para una task que un agent flat con 5 tools resolvía. Resultado: 4 hops de LLM solo para llegar al worker que hace el trabajo real → latencia y costo brutal.

**Cómo lo piensa un senior**: Hierarchical es para **sistemas grandes con dominios disjuntos**. Ejemplo legítimo: agente de soporte de una empresa con 5 productos muy distintos — top-supervisor rutea por producto, cada sub-supervisor maneja workers de su producto. Justificable cuando el costo cognitivo de un supervisor único con 50+ workers supera el costo de la jerarquía. Generalmente NO necesitás más de 2-3 niveles. Si llegás a 4+ niveles, hay un problema de modelado del dominio.

**Tradeoffs reales**:

| Niveles | Cuándo |
|---|---|
| 1 (flat, sin supervisor) | Pocos tools, agente único alcanza |
| 2 (supervisor + workers) | Default multi-agent, 3-10 workers |
| 3 (top + sub-supervisors + workers) | Sistemas con dominios disjuntos claros (multi-producto, multi-departamento) |
| 4+ | Casi siempre sobre-engineering. Reconsiderá el modelado. |

**En entrevista te van a preguntar**:
- Q (mid): *Diferencia entre supervisor flat y hierarchical.*
  A: Flat es 1 supervisor + N workers. Hierarchical anida supervisors — un top delega a sub-supervisors que delegan a sus propios workers. Útil cuando hay dominios disjuntos claros (ej: empresa con departamentos independientes) y un único supervisor con todos los workers sería intratable.
- Q (senior): *¿Cuándo conviene hierarchical sobre flat?*
  A: Cuando (a) tenés ≥3 dominios funcionales claramente disjuntos (no overlap), (b) cada dominio tiene ≥3 sub-workers especializados, (c) el costo de cargar el contexto de TODOS los workers en el supervisor flat hace que el prompt sea inmanejable, (d) querés ownership claro por dominio (cada team mantiene su sub-tree). Si nada de esto aplica, flat gana — menos hops, menos costo.
- Q (trampa, system design): *¿Cómo manejás un caso que cruza dominios en hierarchical (ej: un ticket que necesita producto A + producto B)?*
  A: Trampa común — hierarchical asume disjointness. Cuando hay cross-cutting, opciones: (1) top-supervisor coordina secuencialmente dos sub-supervisors y merge results, (2) crear un "cross-domain worker" specialista en estos casos, accesible desde top, (3) refactorear hacia horizontal-network para esa porción específica. Nunca forzar el patrón cuando no encaja — eso lleva a workarounds frágiles.

### horizontal-network

**Qué es**: Agentes **peer-to-peer**, sin jefe. Coordinan via message passing (cada agente puede llamar a cualquier otro), blackboard pattern (estado compartido donde todos leen/escriben), o protocolos de negociación (votación, debate, consensus).

**La trampa del junior**: Implementar horizontal porque "es más cool" o "más descentralizado". Resultado: deadlocks (A espera a B, B espera a A), infinite loops de delegation, costos imposibles de predecir, debugging miserable.

**Cómo lo piensa un senior**: Horizontal es para casos **donde el control centralizado es genuinamente malo**: (1) **simulaciones** (agent-based modeling de mercados, swarms), (2) **debates / multi-perspective** (varios agentes con roles distintos argumentan, otro arbitra), (3) **research / exploration** (cada agente explora una rama independiente), (4) **resilience** (no querés single point of failure en el supervisor). En aplicaciones de **negocio** comunes, horizontal casi siempre es mala elección — supervisor o hierarchical alcanzan con menos riesgo.

**Tradeoffs reales**:

| Pattern | Pro | Contra |
|---|---|---|
| Supervisor (centralizado) | Control claro, fácil debug | Single point of failure, supervisor bottleneck |
| Hierarchical | Escala mejor, ownership claro | Múltiples hops, latency |
| Horizontal network | Flexibilidad, resilience, fit para simulaciones | Deadlocks, hard to debug, costo difícil de predecir |
| Blackboard (shared state) | Coordinación sin protocolo explícito | Race conditions, state grande |
| Message passing (Actor) | Aislamiento, scalable | Overhead de mensajes |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es un blackboard pattern?*
  A: Patrón donde múltiples agentes leen y escriben en una estructura de datos compartida ("pizarra") en vez de comunicarse directo. Cada agente checkea el blackboard, decide si tiene algo que aportar, y escribe. Otros agentes ven el update y reaccionan. Útil cuando la coordinación es difícil de expresar como protocolo explícito.
- Q (senior): *¿Cuándo justificás horizontal-network sobre supervisor?*
  A: Casos: (1) simulaciones (agent-based modeling), (2) debates/multi-perspective con arbitrer (varios agentes argumentan, uno decide), (3) research donde cada agente explora ramas independientes sin necesidad de coordinación centralizada, (4) sistemas donde quitar el supervisor mejora resilience (no SPOF). En aplicaciones de negocio típicas, NO lo justifico — supervisor gana.
- Q (trampa): *¿Cómo evitás deadlocks en horizontal?*
  A: (1) Timeouts en cada interacción agente-agente. (2) Detección de ciclos en el grafo de calls (si A→B→C→A en una secuencia, abortás). (3) Bounded message passing (cada agente con quota de mensajes outgoing). (4) Pattern de "phases" — todos los agentes deben terminar fase 1 antes de empezar fase 2. (5) Si no podés garantizar acyclic, considerá si horizontal era la elección correcta. La trampa: muchos confían en que "no va a pasar" y deployan sin defenses → bug en producción al mes 2.

### task-delegation

**Qué es**: La **mecánica concreta** de cómo el supervisor (o cualquier delegator) decide a quién delegar: routing prompt vs classifier entrenado vs reglas vs embedding similarity. Y cómo se pasa el "context" al delegado.

**La trampa del junior**: Pasarle TODO el contexto al worker delegado ("acá tenés toda la conversación, todos los results previos, todo"). Worker se confunde con contexto irrelevante, hace peor su tarea. O al revés: pasarle solo la query y NADA de contexto, worker no entiende qué necesita.

**Cómo lo piensa un senior**: Delegation tiene dos preguntas: **(a) quién** y **(b) con qué contexto**. La (b) es la que casi nadie piensa bien. Senior pasa: (1) **task description** clara y específica (no la conversación entera), (2) **context relevante** filtrado (lo que el worker necesita, no más), (3) **expected output format** (qué espera el supervisor de vuelta), (4) **constraints** (límite de tools que puede usar, tiempo, etc), (5) **return reason** ("si no podés hacerlo, decime por qué para que pueda escalarlo"). Esto se llama **handoff prompting** y es arte fino.

**Tradeoffs reales**:

| Routing method | Latencia | Costo | Flexibilidad |
|---|---|---|---|
| LLM con few-shot prompt | Alta (~1-2s) | $$$ | Alta — maneja casos nuevos |
| Embedding similarity (worker description vs query) | Baja (<100ms) | $ | Media — limitado a similitud semántica |
| Classifier fine-tuneado | Muy baja (<50ms) | Casi $0 | Baja — solo casos vistos en training |
| Rule-based (regex/keywords) | Mínima | $0 | Mínima — frágil, no escala |
| Hybrid (classifier + LLM fallback) | Media | $ | Alta — mejor de ambos |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cómo decide un supervisor a qué worker delegar?*
  A: Approach standard: prompt con descripción de cada worker + criterios de routing + la query del usuario + few-shot examples. El LLM emite "delego a worker_X". Alternativas: classifier dedicado, embedding similarity entre query y worker descriptions, o rules. La elección depende de volumen y precisión necesaria.
- Q (senior): *¿Qué contexto le pasás al worker delegado?*
  A: NO todo. Pasás: (1) task description específica destilada por el supervisor (no la conversación cruda), (2) context relevante filtrado, (3) expected output format, (4) constraints. La razón: workers especializados rinden MEJOR con menos contexto irrelevante. Pasar la conversación entera dilute su foco y aumenta costo proporcional a tokens.
- Q (trampa): *El worker recibe la task y dice "no puedo". ¿Qué hace el supervisor?*
  A: Depende de cómo el supervisor está diseñado. (1) Re-routear a otro worker (si hay otro candidato). (2) Pedirle clarification al usuario. (3) Escalar a HITL si está configurado. (4) Devolver "no podemos atender esto" con explicación. (5) Combinar respuestas parciales de varios workers. La trampa: muchos diseños tratan el "no puedo" como crash → user recibe error 500. Senior diseña el supervisor para que SIEMPRE devuelva algo útil al usuario, aunque sea "esto no podemos hacerlo, te derivo a humano".

### conflict-resolution

**Qué es**: Cómo el sistema decide cuando **dos o más agentes proponen acciones/respuestas incompatibles**. Mecanismos: voting (mayoría), debate (agentes argumentan, otro arbitra), confidence-weighted (el de mayor confidence gana), HITL escalation (humano decide), priority rules (worker X siempre gana sobre Y en caso de conflicto).

**La trampa del junior**: No anticipar conflictos. Sistema multi-agent corre divino en el demo (sin conflicto), llega a producción, dos workers proponen acciones contradictorias, el sistema o crashea o ejecuta ambos (con consecuencias raras).

**Cómo lo piensa un senior**: Conflict resolution es parte del **diseño**, no un afterthought. Tiene que estar definido ANTES de deployar: (1) **qué tipos de conflicto pueden surgir** (lista exhaustiva por análisis de modelado), (2) **mecanismo para cada tipo** (no hay un solo "best" — depende del dominio), (3) **fallback final** (siempre debe existir un camino que SIEMPRE resuelve, típicamente HITL o "ningún acción + log para review"), (4) **log de conflictos** para mejora continua. En agents que tocan dinero / acciones irreversibles → conflict resolution explícito + HITL para casos no-claros es **obligatorio**.

**Tradeoffs reales**:

| Mechanism | Cuándo |
|---|---|
| Voting (mayoría simple) | N agentes independientes, decisión categórica |
| Debate + arbiter | Decisiones donde el razonamiento importa más que la respuesta |
| Confidence-weighted | Agentes que reportan confianza calibrada |
| Priority rules | Hay jerarquía clara (security > business logic, por ej.) |
| HITL escalation | Acciones irreversibles, alto stake, baja frecuencia |
| Multi-armed bandit | Querés aprender qué agente acertó más en cada tipo de caso |
| LLM-as-judge | Decisión cualitativa donde un LLM puede arbitrar |

**En entrevista te van a preguntar**:
- Q (mid): *Dos agentes proponen acciones distintas. ¿Cómo resolvés?*
  A: Depende del dominio. Mecanismos comunes: voting (si hay >2), debate con arbiter (si el razonamiento importa), confidence-weighted (si los agentes reportan confianza), priority rules (si hay jerarquía pre-definida), HITL si la acción es irreversible/cara. Lo importante es DECIDIR el mecanismo ANTES de deployar.
- Q (senior): *Estás diseñando un sistema multi-agent que ejecuta trades en bolsa. ¿Cómo manejás conflictos?*
  A: Stack de seguridad obligatorio: (1) priority rules — security/risk agent SIEMPRE gana sobre execution agent. (2) Confidence-weighted con threshold mínimo (si todos los agentes <80% confidence, abortás el trade). (3) HITL para trades >threshold de monto. (4) Circuit breaker: si N conflictos en M minutos, sistema pausa y alerta. (5) Log exhaustivo de cada conflicto para postmortem. (6) Backtesting de los mecanismos con datos históricos antes de deployar real-money.
- Q (trampa): *¿Voting es siempre la opción más fair?*
  A: NO. Voting asume agentes independientes con igual peso, lo cual rara vez es verdad. Si tres agentes están entrenados similarmente, votarán parecido — no es "fair", es ECO. Voting es razonable cuando los agentes son genuinamente diversos (distintos modelos, distintos prompts, distintos contextos). Si no, voting es teatro de coordinación que da false sense of robustness.

## Lo que el libro hace bien acá

- **chapter07** — `Tool Manipulation & Orchestration Agents` — implementa Tool-Using Agent, Chain-of-Agents, y conflict resolution con HITL gates. Buen entry point para entender por qué necesitás resolución de conflictos. Shared memory + state machines básicos.
- **chapter15** — `Education & Knowledge Agents` — el Collective Intelligence con multi-agent consensus es un buen ejemplo concreto de horizontal-network con votación. Vale la pena correrlo para ver el patrón end-to-end.
- **chapter17** — `Epilogue: The Future of Agents` — Emergent Agent Society, Collaboration Spectrum. Más visionario que práctico, pero ayuda a calibrar dónde está la frontera y por qué la mayoría todavía no la cruzó en producción.

## Lo que el libro NO tiene (gaps a saber)

- **Supervisor pattern en LangGraph (production-ready)**: el libro implementa multi-agent en raw Python, no en LangGraph.
  - Recurso: https://langchain-ai.github.io/langgraph/tutorials/multi_agent/agent_supervisor/
  - Qué entender: cómo modelar workers como sub-graphs, supervisor node con conditional edges hacia workers, state compartido con messages reducer, max_iterations control.

- **Hierarchical pattern**: ausente del libro.
  - Recurso: https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/
  - Qué entender: cómo anidar StateGraphs (sub-graph como nodo del top-graph), separación de state por nivel, message passing entre niveles.

- **Anthropic agent patterns paper / Building effective agents**: lectura OBLIGATORIA.
  - Recurso: https://www.anthropic.com/research/building-effective-agents
  - Qué entender: distinción entre workflows (predefined) vs agents (LLM-decides-path), los patterns (prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer), criterios para elegir cada uno. Este paper es el state-of-the-art conceptual de 2024-2025.

- **Multi-agent debate y consensus protocols**: el libro lo toca superficialmente.
  - Recurso: paper "Improving Factuality and Reasoning via Multi-Agent Debate" (Du et al., 2023, MIT). Buscalo en arxiv.
  - Qué entender: cómo orquestar N agentes debatiendo, cuándo el debate mejora factualidad vs ecos, costo computacional realista (3-5x un single agent).

- **AutoGen vs LangGraph para multi-agent**: comparación arquitectónica.
  - Recurso: AutoGen docs (https://microsoft.github.io/autogen/) + comparativas en blogs y Latent Space podcast.
  - Qué entender: AutoGen es más fácil para prototipar multi-agent rápido (groupchat pattern out-of-the-box), LangGraph más control y production-ready. AutoGen tiene mejor tooling para coding agents (research origin).

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `supervisor-pattern`: hacé el tutorial oficial de LangGraph supervisor (link arriba). Implementá un supervisor con 3 workers (researcher, coder, writer). Pegame el grafo + trace de una query que requiere los 3.
- `hierarchical-pattern`: extendé el supervisor anterior agregando un sub-supervisor para "research_team" con 2 workers internos (web_researcher, doc_researcher). Pegame el grafo + explicame qué se hizo más complejo.
- `horizontal-network`: corré `chapter17` o `chapter15` (collective intelligence section). Identificá si es supervisor disfrazado o horizontal real. Justificá tu análisis.
- `task-delegation`: implementá DOS versiones de delegation en el supervisor del primer ejercicio: (a) prompt LLM, (b) embedding similarity con worker descriptions. Compará routing accuracy sobre 20 queries de prueba.
- `conflict-resolution`: armá un mini-sistema donde 3 agentes proponen una respuesta a una query (misma query, 3 LLMs distintos o 3 prompts distintos). Implementá voting + LLM-as-judge para resolver. Compará outputs sobre 5 queries.

### Para subir a `mastered`

- `supervisor-pattern`: en proyecto real, defendé tu decisión de usar supervisor vs single-agent-with-tools con análisis de tradeoffs. Si elegiste single-agent, mostrame cómo respondés a "deberíamos hacerlo multi-agent" en una review técnica. Feynman check: explicáme por qué multi-agent NO siempre es mejor.
- `hierarchical-pattern`: si NO tenés un caso real para hierarchical, defendé por qué no lo necesitás. Si lo tenés, documentá la decisión de niveles + qué dominio maneja cada uno.
- `horizontal-network`: identificá UN caso del mundo real (no del libro) donde horizontal sería superior a supervisor. Argumentá con tradeoffs.
- `task-delegation`: en proyecto real, hacé refactor del prompt de delegation para minimizar context pasado a workers. Medí latencia y costo antes/después.
- `conflict-resolution`: diseñá la matriz de conflict resolution para un sistema real (tipos de conflicto × mecanismo de resolución). Defendela contra "y si pasa X edge case?". Para agents con acciones irreversibles, demostrá el HITL path.

## Anti-patterns que vas a ver en clientes reales

1. **"Vamos a hacerlo multi-agent" sin justificar por qué**
   - Cómo se hace: jefe lee post sobre agentic swarms, pide al equipo "hagamos multi-agent". Equipo implementa por mandate, no por necesidad.
   - Por qué se hace: hype + FOMO + "AI sin multi-agent es viejo".
   - Costo real: 3-5x latencia, 2-10x costo, debugging miserable, equipo demora 3x en deployar features.
   - Cómo lo arregla un senior: empezar con single-agent-with-tools. Solo migrar a multi-agent CUANDO se identifica fricción concreta que multi-agent resuelve (no hipotética). Documentar la decisión.

2. **Supervisor con 15 workers y prompt de 5k tokens**
   - Cómo se hace: agregar workers a medida que surgen casos, el prompt del supervisor crece sin control.
   - Por qué se hace: "lo del último worker que agregué".
   - Costo real: supervisor confunde workers, routing accuracy cae, cada delegation cuesta 5k tokens input.
   - Cómo lo arregla un senior: (a) consolidate workers con overlap, (b) migrar a hierarchical si los workers naturalmente forman grupos, (c) classifier dedicado para routing si volumen lo justifica, (d) revisar si algunos "workers" eran sólo tools (función simple), no agentes.

3. **Sin max_iterations en multi-agent loops**
   - Cómo se hace: dejar que supervisor delegue libremente sin límite.
   - Por qué se hace: "el sistema sabrá cuándo parar".
   - Costo real: en producción aparecen loops (A→B→A→B), cada loop cuesta plata, latency >2min, en peores casos infinite loop hasta que ops mata el proceso.
   - Cómo lo arregla un senior: `recursion_limit` en LangGraph + early-stopping si se detecta repetición + fallback a "no puedo resolver, escalo a humano".

4. **Horizontal network sin timeouts y sin detección de ciclos**
   - Cómo se hace: agentes peer-to-peer llamándose con confianza, "total nos vamos a coordinar bien".
   - Por qué se hace: subestiman complejidad de coordinación distribuida.
   - Costo real: deadlocks ocasionales en producción, requests cuelgan, recovery manual.
   - Cómo lo arregla un senior: timeouts agresivos en cada call agente-agente + detección de ciclos en el grafo de invocaciones + circuit breaker que mata el workflow si supera threshold.

5. **Conflictos resueltos "el último que escribe gana"**
   - Cómo se hace: state shared sin reducer, dos agentes paralelos escriben al mismo campo, el último gana silenciosamente.
   - Por qué se hace: no anticiparon el caso o lo descubrieron tarde.
   - Costo real: bugs no-determinísticos en producción, decisiones perdidas, imposible reproducir.
   - Cómo lo arregla un senior: reducers explícitos en el state schema para campos con escritura concurrente. Conflict resolution explícito (voting, priority, merge function) por campo.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés argumentar **CUÁNDO NO usar multi-agent** con criterios concretos (latencia, costo, complejidad)?
- [ ] ¿Podés diseñar un sistema con supervisor pattern usando LangGraph e incluir defenses (max_iterations, fallback, logging)?
- [ ] ¿Sabés cuándo migrar de supervisor a hierarchical y cuándo eso es over-engineering?
- [ ] ¿Podés explicar tres mecanismos de conflict resolution y elegir el correcto para un dominio dado?
- [ ] ¿Conocés los patterns de "Building Effective Agents" de Anthropic y podés mapear cada uno a casos de uso?
- [ ] En entrevista senior, ¿podés defender single-agent-with-tools contra un interrogador que insiste en multi-agent?
