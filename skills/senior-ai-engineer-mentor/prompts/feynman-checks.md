# Feynman Checks — Pruebas para validar `mastered`

**Cuándo se carga**: lazy, cuando el usuario quiere upgradeear un concepto de `practiced` a `mastered` y necesita pasar el Feynman check.

**Filosofía**: vos dominás algo solo si lo podés EXPLICAR en palabras simples sin jerga. Si tenés que esconderte detrás de términos técnicos, no lo entendés todavía.

**Cómo usar**: para cada concepto, hay un prompt al usuario (la pregunta Feynman), una rúbrica de evaluación (qué demuestra mastery vs recitación), un follow-up de profundidad, y criterios para promover.

**Voz**: Gentleman Rioplatense, exigente pero justo. La explicación tiene que ser CLARA para un no-experto, no rápida.

**Pre-requisito**: el concepto debe estar en `practiced` Y el usuario debe tener un proyecto propio donde lo usó. Sin esos dos, no aplica Feynman check.

---

## react-loop

**Prompt:** "Explicame ReAct en 3 oraciones como si yo fuera un product manager sin background técnico de IA. No uses 'Thought', 'Action', 'Observation', 'loop' ni 'agent'."

**Demuestra mastery si:**
- Usa analogía natural (un asistente que prueba algo, mira el resultado, decide próximo paso)
- Explica el "por qué importa" (versus simplemente preguntar al modelo de una)
- Conecta con experiencia cotidiana del PM

**Solo recita si:**
- Usa la jerga prohibida apenas cambiando palabras
- Explica el "qué" sin el "por qué"
- No puede dar un ejemplo concreto cuando se le pide

**Follow-up:** "Si el LLM no tuviera la capacidad de iterar y solo respondiera 1 vez, ¿qué casos se vuelven imposibles?"

**Decisión:**
- Explicación clara + follow-up sólido → `mastered`
- Explicación clara pero follow-up vago → mantener `practiced`, ofrecer re-check en una semana
- Recitación → mantener `practiced`

---

## json-mode

**Prompt:** "¿Por qué pedirle a un LLM que devuelva JSON puede salir mal? Explicámelo como si yo no supiera qué es JSON."

**Demuestra mastery si:**
- Explica JSON con analogía (una receta con secciones específicas)
- Identifica que "pedirlo" ≠ "garantizarlo"
- Conecta con la idea de validación a nivel de generación

**Solo recita si:**
- Dice "porque el LLM devuelve mal el JSON" sin explicar por qué
- No conecta con la solución (Structured Outputs)

**Follow-up:** "Si en vez de pedir JSON le pedís que devuelva un email, ¿se aplica el mismo problema?"

**Decisión:** la pregunta de follow-up testea generalización — si entiende que es un problema de "garantía de formato" en general, mastery.

---

## function-calling

**Prompt:** "Explicá function calling como si fueras un profesor de secundaria explicando a un estudiante de 16 años qué pasa cuando ChatGPT 'usa' una herramienta."

**Demuestra mastery si:**
- Analogía clara (el LLM "pide" usar la calculadora, alguien la usa, le devuelve el resultado)
- Distingue que el LLM no ejecuta nada — pide
- Explica por qué importa (poder hacer cosas reales en el mundo, no solo texto)

**Recita si:**
- Usa "API", "JSON schema", "endpoint" sin descomponer
- No diferencia "pedir" de "ejecutar"

**Follow-up:** "¿Qué riesgo hay si le decís a un LLM que puede usar la herramienta 'borrar cuenta de usuario'?"

**Decisión:** mastery requiere que el follow-up identifique HITL / least privilege / safety, no solo "puede borrar lo que no toca".

---

## memory-tiers

**Prompt:** "Explicá los 3 tipos de memoria de un agente con una analogía de un asistente humano que conocés bien (ej. una secretaria, un mozo, un médico de familia)."

**Demuestra mastery si:**
- Analogía consistente y completa (las 3 memorias mapean a algo natural)
- Identifica qué se gana con cada tier
- Explica por qué no se puede meter todo en una

**Recita si:**
- Lista los 3 sin analogía o con analogías forzadas
- No puede explicar por qué hay 3 y no 1 o 5

**Follow-up:** "Si tuvieras que elegir un solo tier para sacrificar, ¿cuál y por qué?"

**Decisión:** mastery requiere argumentar la elección, mostrando entender tradeoffs de cada uno.

---

## prompt-patterns

**Prompt:** "Si tuvieras que enseñarle a un colega a escribir un buen prompt en 5 minutos, ¿qué le decís? Nada de PTCF, CoT, ToT — palabras propias."

**Demuestra mastery si:**
- Cubre estructura (quién es el asistente, qué hacer, info necesaria, formato esperado)
- Da ejemplos concretos
- Identifica errores comunes

**Recita si:**
- Listas técnicas sin sustancia
- No puede ejemplificar

**Follow-up:** "¿Cuándo agregar ejemplos al prompt suma y cuándo resta?"

**Decisión:** mastery requiere entender que más ejemplos NO siempre es mejor (overfitting al formato, costo, bias).

---

## chunking-strategy

**Prompt:** "Explicá por qué partir un libro en pedazos para que un LLM pueda buscarlo importa tanto. Que lo entienda alguien que nunca tocó AI."

**Demuestra mastery si:**
- Analogía clara (cómo organizás archivos en una biblioteca)
- Identifica el problema real (chunks malos = no encontrás, encontrás cosas irrelevantes)
- Conecta con calidad downstream

**Recita si:**
- "Chunking importa porque el LLM necesita chunks" — tautológico
- No puede ejemplificar un chunking malo

**Follow-up:** "Si tuvieras que chunkear este libro técnico que tiene tablas, gráficos y texto, ¿qué problemas se te ocurren?"

**Decisión:** mastery requiere identificar 3+ desafíos concretos (layout, tablas, contexto perdido entre capítulos).

---

## embeddings

**Prompt:** "Explicá qué es un embedding usando una analogía con coordenadas geográficas o algo del mundo físico. Sin matemática."

**Demuestra mastery si:**
- Analogía espacial (puntos en un mapa donde cosas parecidas están cerca)
- Explica para qué sirve (encontrar similares, no solo iguales)
- Identifica limitaciones (no captura todo)

**Recita si:**
- Dice "vector denso de N dimensiones" como respuesta completa
- No explica por qué es útil

**Follow-up:** "Si dos frases significan lo mismo pero usan palabras distintas, ¿qué pasa con sus embeddings?"

**Decisión:** mastery requiere conectar el follow-up con "cercanos en el espacio" + entender que es la propiedad central que los hace útiles.

---

## vector-search

**Prompt:** "Explicá vector search a un usuario que conoce búsqueda de Google pero nada más. ¿Qué hace distinto?"

**Demuestra mastery si:**
- Distingue keyword match (Google clásico) vs semantic match
- Da ejemplos donde uno gana sobre el otro
- Conecta con embeddings sin tirar el término al toque

**Recita si:**
- "Busca por similarity de vectores"
- No puede contrastar con Google

**Follow-up:** "¿Cuándo Google clásico (keyword) gana sobre vector search?"

**Decisión:** mastery requiere identificar casos reales (códigos exactos, nombres, jerga técnica) — no quedarse con "vector siempre mejor".

---

## hybrid-retrieval

**Prompt:** "¿Por qué combinar dos métodos de búsqueda en vez de usar el mejor? Explicámelo sin usar 'BM25' ni 'semantic'."

**Demuestra mastery si:**
- Analogía (dos expertos con sesgos distintos opinan mejor que uno)
- Explica qué cubre cada uno
- Identifica el costo (latencia, complexity)

**Recita si:**
- "Combinás keyword + semantic" sin desarrollar
- No identifica downside

**Follow-up:** "Si te da igual la latencia, ¿hybrid siempre gana?"

**Decisión:** mastery requiere reconocer que NO — depende del corpus y queries; medir con eval set.

---

## re-ranking

**Prompt:** "Explicá por qué se hace búsqueda + re-ranking en dos pasos, en vez de uno bueno. Analogía con un proceso humano."

**Demuestra mastery si:**
- Analogía clara (filtro rápido + revisión cuidadosa, ej. CVs)
- Explica tradeoff: speed vs precision
- Identifica que el caro solo se aplica a top-K

**Recita si:**
- "Hay dos modelos"
- No puede explicar por qué dos en vez de uno

**Follow-up:** "¿Por qué no usar el modelo caro desde el principio?"

**Decisión:** mastery requiere entender costo computacional / latencia + escalabilidad.

---

## mcp-protocol

**Prompt:** "¿Qué problema resuelve MCP? Explicalo a un dev backend que sabe REST APIs pero no IA."

**Demuestra mastery si:**
- Analogía con estándares (USB, REST) — un protocolo común evita N x M integrations
- Explica server / client architecture
- Identifica beneficio práctico (reutilización, ecosistema)

**Recita si:**
- "Es el protocolo de Anthropic para tools"
- No identifica el problema de fragmentación

**Follow-up:** "Si ya tenés function calling directo con OpenAI, ¿qué te suma MCP?"

**Decisión:** mastery requiere reconocer cuándo MCP NO suma (single provider, tools internos) — no aplaudirlo siempre.

---

## async-patterns

**Prompt:** "Explicá por qué async importa para llamar 100 LLMs en paralelo. Analogía: cocina de restaurant."

**Demuestra mastery si:**
- Analogía clara (cocinero esperando agua hervida vs preparando otro plato mientras tanto)
- Distingue I/O bound vs CPU bound
- Identifica throughput vs latency

**Recita si:**
- "Async es más rápido"
- No diferencia de threading

**Follow-up:** "¿Por qué no hacer threading en vez de async?"

**Decisión:** mastery requiere conocer tradeoffs reales (overhead, GIL, complexity).

---

## sse-streaming

**Prompt:** "¿Por qué stremear tokens uno por uno en vez de devolver toda la respuesta? Explicá a un PM."

**Demuestra mastery si:**
- Identifica perceived latency
- Compara con experiencia ChatGPT
- Conecta con UX

**Recita si:**
- "Para que vea la respuesta"
- No menciona TTFT

**Follow-up:** "¿En qué caso NO conviene streaming?"

**Decisión:** mastery requiere identificar casos (output que se procesa en bloque, server-to-server interno, parseo JSON).

---

## rate-limits

**Prompt:** "Explicá rate limits y por qué los providers los imponen, a un PM. Analogía con algo del mundo físico."

**Demuestra mastery si:**
- Analogía (tickets por hora en un restaurant, autopistas con peajes)
- Explica protección de infra + fairness
- Identifica qué hacer cuando los hits

**Recita si:**
- "Te limitan los requests"
- No conecta con backoff/retry strategy

**Follow-up:** "Si tenés 5 keys del mismo provider, ¿multiplica el rate limit por 5?"

**Decisión:** mastery requiere matiz — sí pero con coordinación, attribution, riesgo de ToS violation si abusás.

---

## prompt-caching

**Prompt:** "Explicá prompt caching como si le estuvieras vendiendo el feature al CFO. Sin jerga técnica."

**Demuestra mastery si:**
- Analogía financiera (descuento por volumen, repetir compra mismo producto)
- Cuantifica ahorro (~90% en repeated input)
- Identifica cuándo no aplica

**Recita si:**
- "Cachea el prompt"
- No vende el beneficio

**Follow-up:** "Si tu prompt cambia en cada request, ¿qué pasa con el cache?"

**Decisión:** mastery requiere entender que el cache es del prefix estable, no de respuestas — distinción crítica.

---

## cost-optimization

**Prompt:** "Si te pido bajar el costo de un agente a la mitad, ¿cuáles son los primeros 3 pasos sin sacrificar quality? Sin nombres de técnicas — explicá los principios."

**Demuestra mastery si:**
- Mide primero (profile)
- Ataca primero lo más caro
- Verifica que no rompió quality (eval)

**Recita si:**
- Tira técnicas sin método
- No menciona medición/eval

**Follow-up:** "¿Cómo sabés si la optimización rompió la quality?"

**Decisión:** mastery requiere mencionar eval set explícito y A/B comparison.

---

## langchain-basics

**Prompt:** "Explicá qué problema resuelve LangChain a alguien que sabe build con SDK directo del provider. Tradeoffs."

**Demuestra mastery si:**
- Identifica trade between abstraction y control
- Da casos donde LangChain gana
- Da casos donde LangChain pierde

**Recita si:**
- Solo lista features
- No identifica tradeoffs

**Follow-up:** "Un colega quiere migrar todo a LangChain, ¿qué le preguntás antes?"

**Decisión:** mastery requiere ejercicio de senior — preguntar por motivos, scope, eval set existente.

---

## langgraph-dags

**Prompt:** "¿Por qué LangGraph en vez de un loop en Python? Explicalo a un dev senior que no le ve el sentido."

**Demuestra mastery si:**
- Identifica casos donde el loop ad-hoc se rompe (HITL, branching complejo, debugging)
- Conecta con state machines como abstracción
- Reconoce que NO siempre vale (agentes simples)

**Recita si:**
- "Es un grafo de nodos"
- No defiende contra "yo lo hago con un while True"

**Follow-up:** "¿Cuándo NO usar LangGraph?"

**Decisión:** mastery requiere casos claros donde es overkill.

---

## state-management

**Prompt:** "¿Por qué state management en agentes es harder que en una web app tradicional? Explicá a un fullstack dev."

**Demuestra mastery si:**
- Identifica multi-step + persistence + HITL + scope (per-user/session)
- Compara con web stateless vs stateful
- Conecta con costo de fallos (lose conversation history)

**Recita si:**
- "Hay que persistir"
- No identifica peculiaridades

**Follow-up:** "¿Qué se rompe si dos requests pegan al mismo thread_id a la vez?"

**Decisión:** mastery requiere identificar race conditions + estrategia (lock optimista/pesimista/diseño que evita).

---

## checkpointing

**Prompt:** "Explicá checkpointing usando una analogía con un videojuego (saves)."

**Demuestra mastery si:**
- Analogía completa (save points permiten resume, no perdés progreso)
- Conecta con HITL y recovery
- Identifica costo (storage)

**Recita si:**
- "Guarda el state"
- No conecta con casos de uso

**Follow-up:** "Si el agente hizo side effects (mandó emails) entre checkpoints, ¿qué pasa en resume?"

**Decisión:** mastery requiere reconocer idempotency requirement — sin eso, resume = side effects duplicados.

---

## llamaindex-vs-langchain

**Prompt:** "Un equipo está empezando un proyecto RAG. Defendé el caso de cada framework en 2 oraciones cada uno."

**Demuestra mastery si:**
- Cada defensa es honesta (no straw-man)
- Reconoce que pueden coexistir
- Da criterio de decisión claro

**Recita si:**
- "LlamaIndex es para RAG"
- No defiende seriamente ambos

**Follow-up:** "¿En qué proyecto NO usarías ninguno?"

**Decisión:** mastery requiere reconocer cuándo SDK directo gana.

---

## supervisor-pattern

**Prompt:** "Comparalo con una organización de trabajo humana — ¿qué supervisor pattern emula?"

**Demuestra mastery si:**
- Analogía con manager + specialists
- Identifica fallos típicos (micromanager = god object, sin contracts = caos)
- Conecta con cuándo es overkill

**Recita si:**
- "Es un agente jefe"
- No identifica failure modes

**Follow-up:** "¿Cuándo conviene single agent en vez de supervisor?"

**Decisión:** mastery requiere caso claro de NOT — tools homogéneas, app simple, modelo único suficiente.

---

## hierarchical-pattern

**Prompt:** "Por qué jerarquía y no flat — analogía con management corporativo."

**Demuestra mastery si:**
- Justifica span of control (managers no pueden coordinar 50 directos)
- Identifica costos (más overhead, latencia)
- Reconoce cuándo NO escalar

**Recita si:**
- "Es supervisor anidado"
- Sin tradeoffs

**Follow-up:** "¿Cómo decidís cuándo agregar otro nivel?"

**Decisión:** mastery requiere criterios cuantitativos (>N workers, métricas de routing accuracy).

---

## horizontal-network

**Prompt:** "Si un equipo te dice 'vamos a hacer todo horizontal porque es más robusto', ¿qué respondés?"

**Demuestra mastery si:**
- Reconoce ventajas reales (no SPOF, emergent)
- Pone contras claros (unpredictable, hard to test, no SLA)
- Sugiere caso de uso apropiado vs evitar

**Recita si:**
- Acepta o rechaza sin matiz
- No identifica trade

**Follow-up:** "¿Cómo testearías un sistema horizontal?"

**Decisión:** mastery requiere métodos estadísticos + property-based + honesto reconocimiento de límite.

---

## task-delegation

**Prompt:** "Explicá routing en multi-agent system con analogía de un help-desk humano."

**Demuestra mastery si:**
- Analogía clara (call center routing por dept)
- Identifica trade-offs entre rules / classifier / LLM-based
- Reconoce que mal routing = workers ejecutan mal

**Recita si:**
- "El supervisor decide"
- No identifica métodos ni costos

**Follow-up:** "Si tenés 20 workers, ¿qué cambia?"

**Decisión:** mastery requiere estrategia de scale (two-stage, embedding-based, cache).

---

## conflict-resolution

**Prompt:** "Dos agentes proponen acciones opuestas. Explicá 2 maneras de resolverlo + cuándo cada una falla."

**Demuestra mastery si:**
- Distingue voting/arbiter/debate/HITL
- Identifica failure modes (votación con errores correlacionados, arbiter sin info)
- Sugiere diseño que evita conflicts (no solo resolución reactiva)

**Recita si:**
- "Voting"
- Sin failure modes

**Follow-up:** "¿Cómo evitás que los agentes se 'pongan de acuerdo' prematuramente?"

**Decisión:** mastery requiere diversity en prompts/data + medir disagreement entropy.

---

## evals

**Prompt:** "Convencé a un CTO escéptico de invertir en evals. 3 argumentos sin jerga."

**Demuestra mastery si:**
- Cuantifica el riesgo (incidents que evals atrapan)
- Compara con tests de software (analogía conocida)
- Identifica primer paso barato (50 ejemplos)

**Recita si:**
- "Necesitás evals porque sí"
- No vende valor

**Follow-up:** "Si el modelo cambia (Sonnet 4 → Sonnet 5), ¿qué hacés con tu eval set?"

**Decisión:** mastery requiere ejecutarlo full + comparar + decidir si rollout vale, no asumir mejora.

---

## observability

**Prompt:** "¿Por qué observability es harder en agentes que en una API REST tradicional? Explicá a un SRE."

**Demuestra mastery si:**
- Multi-step / multi-agent traces
- LLM behavior es no-deterministic → harder debugging
- Cost tracking adicional
- Drift detection es nuevo concepto

**Recita si:**
- "Tracing"
- No identifica diferencias específicas

**Follow-up:** "Sample 1% de tráfico para eval async — ¿cómo elegís ese 1%?"

**Decisión:** mastery requiere estrategia (random vs stratified, errors 100%, by feature, etc.).

---

## safety-prompt-injection

**Prompt:** "Explicá prompt injection a un security engineer que sabe SQL injection. Paralelos."

**Demuestra mastery si:**
- Paralelo con SQLi (input no confiable mezclado con instrucciones)
- Identifica diferencia (LLM no separa instr/data nativamente)
- Defense in depth como principio compartido

**Recita si:**
- "Es como SQLi"
- Sin paralelos concretos

**Follow-up:** "¿Por qué 'sanitizar el input' no funciona como en SQL?"

**Decisión:** mastery requiere reconocer que el LLM interpreta natural language — no podés escapar instrucciones.

---

## compliance-argentina

**Prompt:** "Explicá a un PM por qué tu app que usa OpenAI desde Argentina tiene un issue de compliance."

**Demuestra mastery si:**
- Identifica Ley 25.326 + transferencia internacional
- Explica qué hace falta (consentimiento + cláusulas)
- Reconoce que ignorar = riesgo regulatorio

**Recita si:**
- "Hay que cumplir la ley"
- Sin específicos

**Follow-up:** "Si un cliente exige data residency AR, ¿qué opciones tenés?"

**Decisión:** mastery requiere mencionar self-host LLM, anonymization pre-LLM, providers con AR region.

---

## compliance-global

**Prompt:** "Tu agent de hiring va a UE. Explicá a un founder qué cambia."

**Demuestra mastery si:**
- Identifica high-risk under AI Act
- Lista obligaciones específicas (risk mgmt, transparency, oversight)
- Identifica timeline + multas como real risk

**Recita si:**
- "GDPR aplica"
- No menciona AI Act específico

**Follow-up:** "¿Qué documentación técnica te van a pedir?"

**Decisión:** mastery requiere mencionar technical documentation, conformity assessment, post-market monitoring.

---

## cost-attribution

**Prompt:** "Explicá cost attribution a un CFO de SaaS B2B."

**Demuestra mastery si:**
- Conecta con chargeback / pricing
- Identifica visibility como prerequisito de optimization
- Sugiere business model (margin sobre LLM cost)

**Recita si:**
- "Tracking de costo por tenant"
- No conecta con business

**Follow-up:** "Tenant excedió budget overnight, ¿qué arquitectura previene esto?"

**Decisión:** mastery requiere mencionar kill switch + alerting temprano + transparency dashboard al tenant.

---

## Decisión post-Feynman

| Resultado | Acción |
|---|---|
| Explicación clara + follow-up sólido | Upgrade a `mastered`, registrar evidence con cita de la explicación |
| Explicación clara pero follow-up débil | Mantener `practiced`, ofrecer re-check |
| Recitación | Mantener `practiced`, sugerir ejercicio para profundizar |
| Mala explicación general | Considerar degradar a `explored` con veto |

Save: `mem_save` con topic_key `skill/ai-engineer-mentor/mastery/{concept}`, en `evidence` citá un fragmento textual de la explicación del usuario que justificó el upgrade. Para auditoría futura.
