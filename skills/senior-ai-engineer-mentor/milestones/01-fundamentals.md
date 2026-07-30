# Hito 1 — Fundamentos + Tool-Use + ReAct

## Por qué importa (perspectiva corporativa)

Acá está el tema: el 80% de los "AI Engineers" que entrevistan en Mercado Libre, Globant, Accenture o cualquier startup con clientes US/EU **NO entienden el loop cognitivo**. Te recitan el código de un agente de LangChain, pero si les preguntás "¿por qué tu agente entra en loop infinito cuando la tool devuelve un error?", se quedan duros. Y sabés por qué? Porque copiaron el snippet sin entender que un agente NO es magia — es un **while loop** con un LLM adentro decidiendo el próximo paso. Si no manejás ese loop, no manejás nada.

Este hito es la base. Sin esto, todo lo que viene — RAG, multi-agente, orquestación — es construir un rascacielos sobre arena. Una empresa que paga 80k USD/año por un AI Engineer Senior no quiere a alguien que sepa instanciar `ChatOpenAI()`. Quiere a alguien que pueda mirar un trace de Langfuse, ver que el agente hizo 17 tool calls cuando debió hacer 3, y diagnosticar si el problema es el prompt, el schema de la function, o el modelo elegido.

Las oportunidades laborales que se abren con este hito sólido: AI Engineer Jr/Mid en empresas con producto LLM en producción (Tinybird, Defined.ai, casi cualquier startup YC W24+), consultor freelance para PYMEs que quieren automatizar con agentes, posiciones de "AI Solutions Engineer" en vendors (Anthropic, OpenAI partners). Sin este hito, ni para junior te llaman de vuelta.

## Conceptos de este hito

### react-loop

**Qué es**: El loop **Thought → Action → Observation** que itera hasta que el agente decide que terminó (Final Answer). Cada iteración: el LLM genera un razonamiento, elige una tool con args, vos ejecutás la tool, le devolvés el resultado, y vuelve a pensar.

**La trampa del junior**: Creer que ReAct es "el algoritmo de los agentes". NO. ReAct es UNA familia de loops — la más simple. Hay Plan-and-Execute (planifica todo upfront, después ejecuta), Reflexion (agrega self-critique), Tree-of-Thoughts (explora ramas). El junior implementa ReAct, ve que funciona en el demo, y se sorprende cuando en producción con observations largos el contexto explota.

**Cómo lo piensa un senior**: ReAct es una **state machine implícita**. Cada iteración modifica un estado (history de pares action/observation). El problema central es: ese state CRECE linealmente, y el LLM tiene que re-leerlo en cada iteración. Latencia y costo crecen O(n²) sobre el número de pasos. Un senior diseña con esto en mente desde el día uno: ¿voy a tener pasos cortos o largos? ¿voy a poder compactar observations? ¿conviene ReAct o un grafo explícito con LangGraph?

**Tradeoffs reales**:

| Loop | Cuándo conviene | Cuándo NO |
|---|---|---|
| ReAct puro | Tareas <10 pasos, observations chicos, exploratorio | Pasos largos, multi-tenant con SLA estricto |
| Plan-and-Execute | Tareas bien estructuradas, sabés el flow upfront | Tareas donde el plan cambia según resultados |
| Reflexion | Tareas donde la calidad importa más que la latencia | Real-time chat |
| LangGraph DAG | Producción seria, necesitás checkpoints/HITL | MVP rápido, demo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es ReAct y por qué se llama así?*
  A: Reasoning + Acting. El LLM alterna entre razonar en lenguaje natural ("Thought") y emitir una acción estructurada ("Action") sobre tools, observando el resultado ("Observation") antes del próximo Thought. La clave es que el razonamiento es VISIBLE en el prompt — eso lo distingue de approaches puramente neuronales.
- Q (senior): *¿Por qué ReAct degrada con observations grandes?*
  A: Porque el history completo se re-envía en cada iteración. Con observations de 5k tokens y 8 pasos, llegás a 40k tokens de input por iteración. Mitigaciones: summarization periódica del history, sliding window, o mover a un grafo explícito donde el state se compacta entre nodos.
- Q (trampa, system design): *Tu agente entra en loop infinito haciendo la misma tool call. ¿Qué hacés?*
  A: Tres causas típicas: (1) la tool devuelve un error que el LLM no entiende como "stop", (2) el prompt no tiene criterio de stop claro, (3) falta un hard limit de iteraciones. Fix: max_iterations + early-stopping basado en repetición de actions + mejora del prompt para que el LLM aprenda a hacer Final Answer cuando la tool falla N veces. La trampa: muchos resuelven solo con max_iterations y dejan al usuario sin respuesta — un senior agrega un fallback de "Final Answer con explicación de la falla".

### json-mode

**Qué es**: Forzar al LLM a generar SOLO JSON válido que matchea un schema (JSON Schema o Pydantic), en vez de texto libre que después parseás con regex y rezás.

**La trampa del junior**: Pedir JSON en el prompt ("Respondé en JSON con esta estructura: {...}") y parsearlo con `json.loads()`. Funciona el 95% de las veces. El otro 5% rompe en producción con un trailing comma, un markdown fence ` ```json `, o un texto explicativo antes del objeto. Y vos te enterás cuando el cliente te escribe a las 3am.

**Cómo lo piensa un senior**: JSON mode NO es una feature de prompting — es una **constraint a nivel de decoding**. OpenAI Structured Outputs y Anthropic tool use usan constrained decoding o grammar-based sampling para garantizar que el output ES válido. La diferencia entre "pedir JSON" y "forzar JSON" es la diferencia entre rezar y tener un contrato.

**Tradeoffs reales**:

| Approach | Garantía | Costo | Compatibilidad |
|---|---|---|---|
| Prompt "respondé en JSON" | 95% — falla raro pero falla | Cero overhead | Cualquier modelo |
| `response_format={"type":"json_object"}` (OpenAI) | JSON válido garantizado, schema NO | Mínimo | OpenAI / compat |
| Structured Outputs con schema (OpenAI) | JSON válido + schema garantizado | ~10% latencia extra | gpt-4o, gpt-4o-mini |
| Tool use schema (Anthropic) | Schema garantizado vía tool | Cero extra (es tool) | Claude 3+ |
| Outlines / Guidance (open-source) | Garantía total con grammar | Local, depende del backend | Llama, Mistral |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cuál es la diferencia entre pedir JSON en el prompt y usar JSON mode?*
  A: Prompt es probabilístico, JSON mode es constrained. Con prompt el modelo PUEDE generar tokens inválidos (cierre faltante, comilla mala). Con JSON mode el sampler sólo elige entre tokens que mantienen el JSON sintácticamente válido — es imposible que falle el parse.
- Q (senior): *¿Por qué Structured Outputs de OpenAI agrega latencia y JSON mode "simple" no?*
  A: Structured Outputs primero compila el schema a una grammar/state machine y constraint mask. Esa compilación tiene overhead la primera vez (cacheado después). JSON mode simple solo restringe a "token válido para JSON genérico" — más rápido pero no garantiza el schema, solo la sintaxis.
- Q (trampa): *Tu schema tiene un campo opcional `enum`. El modelo a veces devuelve `null`, a veces lo omite, a veces inventa un valor fuera del enum. ¿Es bug del modelo o tuyo?*
  A: Tuyo. Si usás Structured Outputs con el enum bien declarado, es imposible que devuelva fuera del enum. "Omite el campo vs `null`" depende de si lo marcaste `required` o no en el schema. La trampa: muchos culpan al modelo cuando el schema está mal especificado.

### function-calling

**Qué es**: El protocolo por el cual el LLM "pide" ejecutar una función con args tipados (en formato JSON), tu runtime ejecuta la función real, y le devolvés el resultado al modelo para que siga.

**La trampa del junior**: Pensar function calling como "el modelo llama mi función". NO LA LLAMA. El modelo emite un MENSAJE que dice "quiero llamar a `get_weather` con `{city: 'BA'}`". Vos sos el que ejecuta. El modelo nunca toca tu código. Esta confusión lleva a malas decisiones de seguridad (creer que el modelo "está sandboxed" cuando en realidad VOS sos el sandbox).

**Cómo lo piensa un senior**: Function calling es **un schema de mensajes**. Es la capa de transporte. Lo importante son las **políticas alrededor**: qué tools exponés, con qué argumentos válidos, qué hace tu ejecutor con esos args (validación, rate limiting, auth, audit log). El modelo es untrusted input — siempre. Cada arg que recibís de una function call debe pasar las mismas validaciones que un body de POST público.

**Tradeoffs reales**:

| Approach | Pro | Contra |
|---|---|---|
| OpenAI function calling clásico | Maduro, multi-tool, parallel calls | Vendor-specific format |
| Anthropic tool use | Mejor para razonamiento extendido, tool_use blocks visibles | API distinta |
| MCP (Model Context Protocol) | Estándar abierto, server-side discovery, multi-provider | Más infra (servidor MCP) |
| ReAct-style "Action: tool(args)" en texto | Funciona con cualquier modelo | Frágil al parse, sin types garantizados |

**En entrevista te van a preguntar**:
- Q (mid): *¿Quién ejecuta la función en function calling?*
  A: Tu aplicación. El LLM solo emite un mensaje estructurado que indica nombre de función y args. El modelo no tiene acceso al runtime, no hay sandbox automático. Esto es feature, no bug — vos controlás qué se ejecuta.
- Q (senior): *¿Cómo manejás parallel tool calls?*
  A: GPT-4o y Claude 3.5+ pueden devolver MÚLTIPLES tool_use en un solo turno. Las ejecutás en paralelo (asyncio.gather), recolectás los resultados, y los devolvés todos en el siguiente turno como tool_result blocks. Cuidado con tools que tienen side effects o conflicts — necesitás idempotencia o un mutex lógico.
- Q (trampa): *Te llega una function call con `{user_id: "'; DROP TABLE users; --"}`. ¿De quién es el problema?*
  A: Tuyo, 100%. El LLM puede ser inducido vía prompt injection a generar args maliciosos. Tu executor DEBE validar cada arg con el mismo rigor que validás un input HTTP — schema validation, escape de SQL, principle of least privilege en la conexión DB. El modelo es input untrusted.

### memory-tiers

**Qué es**: Tres tipos de memoria que un agente serio maneja: **working** (turno actual, contexto inmediato), **episodic** (eventos pasados — qué hizo el usuario, qué tools se llamaron), **semantic** (conocimiento estable — perfil del usuario, hechos del dominio, embeddings indexados).

**La trampa del junior**: Mandar todo el history en cada turno y rezar que el context window aguante. O peor: usar "memoria" como sinónimo de "vector store con embeddings de los últimos N mensajes". Eso NO es memoria episódica, es retrieval pobre.

**Cómo lo piensa un senior**: Cada tier tiene **storage, refresh strategy y latency budget distintos**. Working memory vive en el prompt (caro por token). Episodic vive en un store estructurado (Postgres, Redis) con queries específicas ("¿qué decisiones tomó este user los últimos 7 días?"). Semantic vive en un vector store o knowledge graph (refresh batch, eventual consistency aceptable). Confundirlos te lleva a meter logs de eventos en vector store y pagar 4x más por queries que un SELECT con WHERE resuelve mejor.

**Tradeoffs reales**:

| Tier | Storage típico | Latencia objetivo | Refresh | Cuándo falla |
|---|---|---|---|---|
| Working | Prompt context | <100ms (es local) | Por turno | Context overflow, costo |
| Episodic | Postgres/Redis | <50ms | Real-time write | Schema rígido si el dominio cambia |
| Semantic | Vector DB / KG | <200ms (top-K) | Batch o event-driven | Stale embeddings, schema drift |

**En entrevista te van a preguntar**:
- Q (mid): *Diferencia entre memoria episódica y semántica.*
  A: Episódica = eventos con timestamp ("el user pidió X el martes"). Semántica = conocimiento atemporal ("el user prefiere respuestas cortas"). La episódica responde "qué pasó", la semántica responde "qué es verdad".
- Q (senior): *Tu agente recuerda preferencias del usuario. ¿Dónde las guardás y por qué?*
  A: Depende de cuántas son y cuánto cambian. Pocas y estables → semantic en una tabla `user_preferences` con cache. Muchas y evolutivas → embedding del perfil con refresh batch nocturno. Nunca en working memory (se pierde al cerrar sesión) ni todas en episodic (queries lentas).
- Q (trampa, system design): *Tu agente "olvida" lo que dijo el user hace 3 turnos. ¿Cómo diagnosticás?*
  A: Tres causas: (1) truncaste el history por límite de tokens (mirá si el sliding window cortó esos turnos), (2) compactaste con summarization y perdiste detalle (revisá el summary), (3) episodic no escribió por error (revisá logs del write). La trampa: muchos suben el max_tokens sin diagnosticar y pagan 3x sin arreglar nada.

### prompt-patterns

**Qué es**: Familia de patterns para estructurar prompts: **PTCF** (Persona-Task-Context-Format), **Chain-of-Thought** (CoT, "pensá paso a paso"), **Tree-of-Thoughts** (ToT, explorá ramas), **Few-Shot** (mostrale ejemplos), **Self-Consistency** (sampleá N respuestas y votá).

**La trampa del junior**: Mezclar todos los patterns sin criterio ("le pongo persona, le pido que piense paso a paso, le doy 8 ejemplos, y le pido JSON") y terminar con prompts de 4k tokens que cuestan una fortuna y son frágiles. O al revés: usar zero-shot para todo y quejarse de que el modelo es "tonto".

**Cómo lo piensa un senior**: Cada pattern resuelve **un problema distinto**. CoT mejora razonamiento matemático/lógico pero NO mejora classification simple. Few-Shot baja varianza pero sube costo de input. ToT es caro (N caminos) y solo vale cuando el costo de error es alto. Un senior elige basado en: ¿qué tipo de error está cometiendo el modelo? Y mide A/B antes de comprometerse.

**Tradeoffs reales**:

| Pattern | Sube | Baja | Cuándo usar |
|---|---|---|---|
| PTCF | Consistencia | Brevedad | Production agents con rol claro |
| CoT | Reasoning accuracy | Latencia, costo | Math, lógica, multi-step planning |
| Few-Shot (3-5 ej) | Format adherence, edge cases | Tokens input | Tareas con format específico o ambiguas |
| ToT | Calidad en problemas complejos | Costo ×N | Pocas decisiones críticas (no real-time) |
| Self-Consistency | Robustez | Costo ×N llamadas | Evals, no producción típica |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es Chain-of-Thought y cuándo NO usarlo?*
  A: Pedirle al modelo que verbalice su razonamiento antes de la respuesta final. Mejora tareas de lógica/math. NO usarlo en classification simple (overhead sin beneficio), tareas de generación creativa (introduce sesgo), o cuando necesitás latencia bajísima.
- Q (senior): *Tenés un agente que clasifica tickets en 12 categorías. Probaste zero-shot, CoT y few-shot. CoT tira mejor accuracy pero 3x más latencia. ¿Qué hacés?*
  A: Análisis costo/beneficio: ¿el delta de accuracy compensa la latencia para el SLA? Si es clasificación interna (humano revisa después), zero-shot probablemente alcanza. Si es routing automático con downstream caro, CoT vale. Alternativa: fine-tunear un modelo chico con outputs CoT del modelo grande (distillation) — tenés la calidad sin la latencia.
- Q (trampa): *Few-shot con 10 ejemplos vs 3 ejemplos: ¿siempre mejor 10?*
  A: NO. Más ejemplos = más tokens input = más caro y más lento. Y a partir de cierto N el marginal benefit cae a cero o se vuelve NEGATIVO (el modelo se "obsesiona" con los ejemplos y pierde generalización). Empezá con 3, medí, agregá si vale.

## Lo que el libro hace bien acá

- **chapter01** — `Foundations of Agent Engineering` — el notebook implementa un ReAct loop minimal con MockLLM y muestra los tres tipos de agent brain (Reactive/Deliberative/Hybrid). Excelente para ver el while loop pelado SIN frameworks. Corrélo y modificá el `max_iterations` para ver cómo cambia el comportamiento.
- **chapter03** — `The Art of Agent Prompting` — cubre PTCF, Chain-of-Thought, Tree-of-Thought, Few-Shot con ejemplos comparativos entre 4 providers. La sección de A/B testing de prompts es oro para entender por qué medir importa más que intuir.
- **chapter05** — `Foundational Cognitive Architectures` — implementa memory tiers (working/episodic/semantic) con un agente que recuerda eventos entre sesiones. El código de `MemoryAugmentedAgent` muestra bien la separación de stores.
- **chapter08** — `Data Analysis & Reasoning Agents` — el `GeneralProblemSolver` con 5-stage meta-reasoning es buen ejercicio para ver Plan-and-Execute en acción (no es ReAct puro, contrastá).

## Lo que el libro NO tiene (gaps a saber)

- **JSON mode / Structured Outputs profundo**: el libro lo menciona pero no entra al constrained decoding ni a Structured Outputs de OpenAI.
  - Recurso: https://platform.openai.com/docs/guides/structured-outputs
  - Qué entender: la diferencia entre `response_format: json_object` (sintaxis garantizada) y `response_format` con schema (sintaxis + estructura garantizadas). La compilación del schema, el caching de la grammar, y por qué los `enum` con valores únicos siempre se respetan.

- **Anthropic tool use con tool_choice forced**: el libro usa function calling estilo OpenAI; Anthropic tiene matices propios.
  - Recurso: https://docs.anthropic.com/en/docs/build-with-claude/tool-use
  - Qué entender: tool_use blocks vs tool_result blocks, parallel tool use, `tool_choice: {type: "tool", name: "X"}` para forzar una tool específica (útil para extract-from-text patterns).

- **Constrained decoding open-source**: Outlines, Guidance, LMQL — cómo forzar grammars con modelos locales (Llama, Mistral).
  - Recurso: https://github.com/dottxt-ai/outlines
  - Qué entender: el concepto de grammar/regex sobre el sampler, costo computacional, y cuándo conviene vs llamar a un modelo cerrado con Structured Outputs.

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `react-loop`: corré `chapter01/notebook.ipynb`, ejecutá el `ReactiveAgent` y modificá `max_iterations` de 5 a 2 y a 20. Pegame el output de los tres runs y explicame qué cambia y por qué.
- `json-mode`: NO hay notebook directo. Implementá un script de 30 líneas que: (a) pida al modelo una respuesta en JSON con prompt-only, (b) la misma con `response_format=json_object`, (c) la misma con Structured Outputs y schema. Corré los 3 con un prompt adversarial ("respondé como pirata, después dame el JSON"). Mostrame qué falla en cada caso.
- `function-calling`: corré `chapter01` o `chapter07` con un agente tool-using. Agregá una tool nueva con args tipados. Validá que el agente la usa correctamente.
- `memory-tiers`: corré `chapter05/notebook.ipynb` y mostrame en qué estructura de datos vive cada tier. Identificá dónde está working vs episodic vs semantic.
- `prompt-patterns`: corré `chapter03` y compará el output de PTCF vs zero-shot sobre el mismo task. Mostrame el delta.

### Para subir a `mastered`

- `react-loop`: implementá un agente ReAct sobre un dominio tuyo (no del libro) — por ejemplo, un agente que busca info de productos en una API real. Explicáme en 3 oraciones por qué ReAct degrada cuando el observation context crece, y cómo lo mitigarías en producción.
- `json-mode`: en un proyecto real, reemplazá un parser regex de output del LLM por Structured Outputs. Medí: tasa de error antes vs después, latencia delta, costo delta. Defendé la decisión con números.
- `function-calling`: diseñá el set de tools de un agente real (mínimo 5 tools). Justificá cada arg, su validación, y qué pasa si el LLM manda args inválidos. Feynman check: explicáselo a alguien que no sabe AI en 5 minutos sin tecnicismos.
- `memory-tiers`: en un proyecto propio, definí explícitamente qué va en cada tier, dónde vive, cuál es el refresh strategy. Hacé un diagrama. Defendelo.
- `prompt-patterns`: corré un A/B test real entre 2 patterns sobre la misma task. Mostrame metric, sample size, y la decisión final con justificación.

## Anti-patterns que vas a ver en clientes reales

1. **"El agente entra en loop infinito" sin max_iterations**
   - Cómo se hace: copy-paste del tutorial de LangChain sin tocar `max_iterations=15` default.
   - Por qué se hace: "funcionó en el demo, no toqué nada".
   - Costo real: agente corre 15 iteraciones sobre un error trivial, costo por request explota 10x, user espera 40 segundos.
   - Cómo lo arregla un senior: max_iterations explícito según task type + early-stopping basado en repetición + fallback a Final Answer con mensaje de error humano.

2. **Parsear JSON con regex porque "JSON mode es muy nuevo"**
   - Cómo se hace: `re.findall(r'\{.*\}', response, re.DOTALL)` y rezar.
   - Por qué se hace: lo aprendieron pre-2024, no actualizaron mental model.
   - Costo real: 2-5% de requests fallan parse en producción, bugs intermitentes imposibles de reproducir, soporte loco.
   - Cómo lo arregla un senior: Structured Outputs o tool use schema. Fin.

3. **Mandar todo el conversation history en cada turno sin compactar**
   - Cómo se hace: `messages.append(...)` infinito, mandar a la API.
   - Por qué se hace: "el modelo necesita contexto".
   - Costo real: a los 20 turnos pagás 3x; a los 50 explotás context window y la app revienta.
   - Cómo lo arregla un senior: sliding window + summarization periódica + persist en episodic memory para queries específicas. El context del prompt no es la única memoria.

4. **No validar args de function calls**
   - Cómo se hace: `db.query(args['sql'])` directo desde el tool result.
   - Por qué se hace: "el modelo es smart, no genera SQL injection".
   - Costo real: prompt injection del usuario te abre la base entera. Caso real: Samsung 2023, code leak vía ChatGPT.
   - Cómo lo arregla un senior: schema validation con Pydantic + parameterized queries + least-privilege DB user + audit log.

5. **Few-shot con 15 ejemplos "por las dudas"**
   - Cómo se hace: meter todos los edge cases que se les ocurrió como few-shot.
   - Por qué se hace: "más ejemplos = mejor".
   - Costo real: prompt de 6k tokens input por request, costo lineal en cada call, latencia +200ms.
   - Cómo lo arregla un senior: 3-5 ejemplos curados que cubren los tipos distintos. Si necesitás más, es problema de fine-tuning o de patron mal elegido.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés implementar un ReAct loop desde cero (sin LangChain) en <50 líneas y explicar dónde está el riesgo de loop infinito y context overflow?
- [ ] ¿Podés explicar la diferencia entre "pedir JSON en el prompt" y Structured Outputs, y en qué casos cada uno es la decisión correcta?
- [ ] Si te dan un agente con 8 tools en producción, ¿podés listar 5 cosas que validás antes de confiar en los args que el LLM emite?
- [ ] ¿Podés diseñar la memoria de un agente conversacional dividiendo working/episodic/semantic y justificar el storage de cada tier?
- [ ] En una entrevista senior, ¿podés defender por qué usaste CoT vs zero-shot vs few-shot en un proyecto específico, con métricas?
