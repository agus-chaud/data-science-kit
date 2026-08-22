# Onboarding Bootstrap — Protocolo de 4 minutos

**Cuándo ejecutar**: cuando `mem_search query="skill/ai-engineer-mentor/mastery"` no devuelve ninguna entrada para los 36 conceptos del catálogo. Una sola vez por usuario, salvo `/ai-mentor reset`.

**Cuándo NO ejecutar**: si ya hay aunque sea UNA entrada de mastery → asumí que el usuario ya pasó por bootstrap.

**Voz**: Gentleman Rioplatense, voseo, cálido. Sin ceremonia exagerada — somos eficientes.

---

## Step 1 — Greeting (10 segundos)

Decí exactamente esto (adaptando el saludo al contexto del mensaje original):

> *Loco, antes de tirarme con la respuesta — esta es la primera vez que activás el mentor de AI Engineering, así que dame 4 minutos para calibrarte. Sin esto te trato como Junior o como Senior sin saber cuál sos, y cualquiera de las dos te hace perder el tiempo. ¿Vamos?*

Esperá confirmación corta (sí / dale / vamos). Si dice "no, después" → respondé la consulta original SIN bootstrap, pero NO grabes mastery (queda `unknown` implícito y se vuelve a ofrecer la próxima).

---

## Step 2 — La misión (30 segundos, 1 pregunta)

ANTES de los probes, capturá el PARA QUÉ. Sin misión, todo lo que enseñe se siente abstracto. Una sola pregunta:

> *Antes de medirte: ¿para qué querés esto? Decime el rol/empresa o mercado que apuntás y en qué plazo. Algunos ejemplos para que te orientes:*
>
> *- "RAG Engineer en fintech, clientes EU, en 6 meses estar entrevistando"*
> *- "Multi-agent Engineer en una startup, mercado global, sin plazo fijo, por gusto"*
> *- "AI Solutions Architect corporativo en LATAM, 1 año"*
> *- "Todavía no sé el rol exacto, vengo de backend y quiero meterme en AI en general"*

De la respuesta extraé `target_role`, `domain`, `market`, `timeline` y `why`. Si algún campo no lo dice, dejalo en `general`/`sin definir` — NO interrogues de más, esto es 1 pregunta, no un cuestionario. Guardá la misión en engram (ver Step 6).

> Esto NO suma tiempo al onboarding: reemplaza ceremonia por foco. Es 1 pregunta de ~30 segundos.

---

## Step 3 — Background calibration (1 minuto)

Hacé esta pregunta ÚNICA con 4 opciones:

> *Primera y única pregunta de contexto: ¿de dónde venís hoy con AI Engineering?*
>
> *1. **Junior** — toqué algún ChatGPT API, leí algo de prompts, no construí agentes todavía*
> *2. **Mid** — armé chatbots con RAG básico, manejo embeddings y vector DB, no me metí con multi-agente serio*
> *3. **Senior** — construí agentes en producción, conozco LangChain/LangGraph, evals, observability, vine a profundizar y cubrir gaps*
> *4. **Cambio de stack** — vengo de otro mundo (frontend/backend/data/devops/etc.) y arranco en AI*

Esperá respuesta. Según el número, aplicá este **bootstrap pattern**:

### Opción 1 — Junior
- **Todos los 36 conceptos** (incluye Hito 0) → `unknown`
- `evidence: "bootstrap-junior-{YYYY-MM-DD}"`

### Opción 2 — Mid
- `explored`: `react-loop`, `json-mode`, `function-calling`, `prompt-patterns`, `chunking-strategy`, `embeddings`, `vector-search`, `langchain-basics`, `async-patterns`, `rate-limits`
- `unknown`: el resto (26 conceptos, incluye los 4 de Hito 0 — integración práctica es un skill aparte de "sabe usar function-calling en un notebook")
- `evidence: "bootstrap-mid-{YYYY-MM-DD}"`

### Opción 3 — Senior
- **Todos los 36 conceptos** → `explored`
- `evidence: "bootstrap-senior-claimed-{YYYY-MM-DD}"`
- *Nota interna*: los probes del Step 3 son ESPECIALMENTE importantes acá — si fallan, bajás varios a `unknown` ANTES de guardar.

### Opción 4 — Cambio de stack
Preguntá una sub-pregunta:
> *¿De qué stack venís? (frontend / backend / data / devops / móvil / otro)*

Según respuesta:
- **Backend / DevOps**: `explored` para `async-patterns`, `rate-limits`, `observability`, `cost-optimization`, `cost-attribution`. Resto `unknown`.
- **Data / ML**: `explored` para `embeddings`, `vector-search`, `chunking-strategy`, `evals`, `observability`. Resto `unknown`.
- **Frontend**: `explored` para `sse-streaming`, `async-patterns`. Resto `unknown`.
- **Móvil / Otro**: todos `unknown`.
- `evidence: "bootstrap-stackswitch-{stack}-{YYYY-MM-DD}"`

---

## Step 4 — Conceptual probes (2 minutos, 3 preguntas)

Independientemente de la opción anterior, hacé estos 3 probes para refinar. Tono Gentleman, sin embromar — son diagnósticos rápidos.

### Probe 1 — `react-loop`

> *Describime ReAct en 2 oraciones. Tu propia versión, no me cites Wikipedia.*

**Lectura de respuesta**:
- **Buena** (menciona loop Thought→Action→Observation + razón por la cual itera): mantené `react-loop` en el nivel de bootstrap o subilo a `explored` si estaba `unknown`.
- **Media** (menciona "razona y actúa" pero sin el loop iterativo o sin observation): `explored` si estaba `practiced/mastered`, sino dejá `unknown`.
- **Mala / "no sé"**: forzá `react-loop` a `unknown`, independientemente del bootstrap.

### Probe 2 — RAG architecture (cubre `chunking-strategy` + `vector-search` + `hybrid-retrieval`)

> *Si te dan 1M de documentos heterogéneos (PDFs, HTMLs, transcripts) y te piden armar un RAG corporativo, ¿por dónde arrancás? Dame los primeros 3 pasos.*

**Lectura**:
- **Buena** (menciona estrategia de chunking pensada, embeddings con elección de modelo, vector store + alguna mención de hybrid o re-ranking): mantené los 3 conceptos en bootstrap level.
- **Media** (menciona embeddings + vector DB pero sin pensar chunking ni re-ranking): bajá `hybrid-retrieval` y `re-ranking` a `unknown`.
- **Mala** ("uso LangChain con FAISS y listo"): bajá los 3 a `unknown` y `chunking-strategy` también.

### Probe 3 — frameworks (`langchain-basics` + `langgraph-dags`)

> *Diferencia entre LangChain y LangGraph en 1 oración. ¿Cuándo usás cuál?*

**Lectura**:
- **Buena** (LangChain = chains/orquestación lineal o LCEL; LangGraph = grafos de estado para control de flujo cíclico, multi-agente, HITL): mantené ambos en bootstrap level.
- **Media** ("LangGraph es para grafos" sin entender por qué): bajá `langgraph-dags` a `unknown`.
- **Mala / "son lo mismo" / "no conozco LangGraph"**: bajá ambos a `unknown`.

---

## Step 5 — Dirección (30 segundos)

Después de los probes, decí:

> *Listo, te calibré. Tenés mastery state guardado, así que de ahora en adelante no te pregunto de nuevo nada de esto — me ajusto solo. ¿Por dónde arrancás?*
>
> *a) **Hito 1** desde el principio (recomendado si bootstrappeaste Junior)*
> *b) **El concepto donde te trabaste en el probe** (te lo destrabo ya)*
> *c) **Tu pregunta original** que veníamos a responder antes del bootstrap*
> *d) **Algo distinto** — decime qué*

Esperá respuesta y proceder.

---

## Step 5.5 — Mapa rápido: cómo usarme (SIEMPRE, no saltear)

Antes de proceder con la dirección elegida, mostrale al usuario ESTA tabla compacta — es la parte que más
se salta en onboardings típicos y la que más dudas genera después ("¿esto lo activo yo o se activa solo?").
Adaptala según si detectaste contexto de TP/proyecto de integración (mission.domain lo sugiere, o el
usuario lo mencionó, o el proyecto activo tiene forma de integración práctica — webhook, bot, agente sobre
una plataforma externa).

> *Antes de arrancar, 30 segundos de "cómo usarme" — para que no tengas que redescubrirlo a los ponchazos:*
>
> *- **Preguntame como a un compañero senior** — "no entiendo X", "diferencia entre X e Y", "cuándo uso X" — me activo solo, no hace falta comando.*
> *- **`interview {concepto}`** — cuando querés que te interrogue en serio, modo hostil-justo, para prepararte para entrevista.*
> *- **`review` + tu código** — te lo destrozo quirúrgicamente, sin piedad y sin vueltas.*
> *- **`project: {tu idea}`** — te armo un plan de MVP realista, con stack opinado y mapeo a lo que ya sabés. **Si tu idea todavía es una idea sin forma** (no un scope claro), te voy a mandar primero a `/office-hours` para el brainstorming — ese es su trabajo, no el mío.*
> *- **`explain {paper|repo}`** — te leo una arquitectura ajena y te la traduzco.*
> *- **`/ai-mentor status`** / **`/ai-mentor next`** — en cualquier momento, para ver dónde estás parado o qué sigue.*

**Si el contexto es un TP/proyecto de integración práctica (webhook + agente sobre una plataforma externa,
tipo bot conversacional que guarda datos)**, agregá esta línea — es la que más tiempo ahorra:

> *Para tu tipo de proyecto específicamente: ya tengo un **Hito 0** armado (integración práctica — webhook,
> memoria de agente, validación de tool-calling, patrones de auth) y `project: {tu idea}` reconoce esta
> forma de proyecto y te precarga el scope típico en vez de arrancar de cero. Usalo apenas tengas el scope
> claro — si todavía no lo tenés, `/office-hours` primero.*

No conviertas esto en un cuestionario — es informativo, una sola pasada, sin pedir confirmación de cada
punto. Seguí directo a Step 6.

---

## Step 6 — Guardar bootstrap en engram (CRÍTICO, no saltar)

### 6a — La misión

Primero, persistí la misión capturada en Step 2:

```yaml
topic_key: skill/ai-engineer-mentor/mission
type: preference
project: 30-agents-every-ai-engineer-must-build  # o el proyecto activo del usuario
title: "Misión del usuario: {target_role}"
content: |
  target_role: "{ej. RAG Engineer, Multi-agent Engineer, AI Solutions Architect}"
  domain: "{ej. fintech, e-commerce, health, general}"
  market: "{es-AR | global | latam | mixed}"
  timeline: "{ej. 6 meses para estar entrevistando | sin definir}"
  why: "{1-2 oraciones — la razón real}"
  updated: "{YYYY-MM-DD}"
```

### 6b — Mastery por concepto

Para cada uno de los 36 conceptos en `concepts.md`, llamá `mem_save` con:

```yaml
topic_key: skill/ai-engineer-mentor/mastery/{concept-slug}
type: pattern
project: 30-agents-every-ai-engineer-must-build  # o el proyecto activo del usuario
title: "Mastery bootstrap: {concept-slug} = {level}"
content: |
  **What**: Bootstrap inicial del concepto `{concept-slug}` en nivel `{level}`.
  **Why**: Onboarding de la skill senior-ai-engineer-mentor, opción {1-4} ({junior/mid/senior/stackswitch}).
  **Where**: skill engram namespace.
  **Learned**: {nota del probe si aplica — ej. "Probe react-loop fallido, forzado a unknown" o "Probe RAG correcto, mantenido en explored"}
  ---
  level: {unknown | explored | practiced | mastered}
  evidence: "bootstrap-{opcion}-{YYYY-MM-DD}"
  last_validated: "{YYYY-MM-DD}"
  next_review: "{YYYY-MM-DD calculado: last_validated + intervalo base del nivel; unknown → sin fecha}"
  ease: 2.0
  misconceptions: ["{si un probe reveló una creencia errónea específica + fecha}"]
  history:
    - "{YYYY-MM-DD}: bootstrap from option {N} ({label})"
```

**Optimización**: si el bootstrap es 100% `unknown` (opción 1 sin variaciones de probe), podés guardar UNA observación resumen `skill/ai-engineer-mentor/mastery/_bootstrap-snapshot` con la lista completa, y crear entries individuales solo a medida que cada concepto se trabaja. La skill puede leer el snapshot como fallback.

**Importante**: guardá TAMBIÉN una entrada índice para que futuras búsquedas la encuentren rápido:

```yaml
topic_key: skill/ai-engineer-mentor/prefs/bootstrap-done
title: "Bootstrap completado"
content: |
  **What**: Onboarding 4-min completado, opción {N}.
  **When**: {YYYY-MM-DD}
  **Concepts initialized**: 36
```

---

## Edge cases

- **Usuario dice "no me hagas el bootstrap, andá directo"** → respondé la consulta original, NO guardes mastery, ofrecé `/ai-mentor` para hacerlo después.
- **Usuario responde con bronca o evasivo a los probes** → cortá los probes y bootstrappeá conservador (un nivel menos que el background declarado).
- **Usuario salta de opción a opción** → quedate con la última y procedé.
- **Si engram falla (`mem_save` da error)** → avisá: *"No pude persistir tu mastery state, voy a tener que repreguntarte la próxima. Igual seguimos."*
