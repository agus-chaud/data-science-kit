# Onboarding Bootstrap — Protocolo de 4 minutos

**Cuándo ejecutar**: cuando `mem_search query="skill/data-engineer-mentor/mastery"` no devuelve ninguna
entrada para los 36 conceptos del catálogo. Una sola vez por usuario, salvo `/de-mentor reset`.

**Cuándo NO ejecutar**: si ya hay aunque sea UNA entrada de mastery → el usuario ya pasó por bootstrap.

**Voz**: Gentleman Rioplatense, voseo, cálido. Sin ceremonia exagerada — somos eficientes.

---

## Step 1 — Greeting (10 segundos)

Decí esto, adaptando el saludo al contexto del mensaje original:

> *Loco, antes de tirarme con la respuesta — esta es la primera vez que activás el mentor de Data
> Engineering, así que dame 4 minutos para calibrarte. Sin esto te trato como junior o como senior sin saber
> cuál sos, y cualquiera de las dos te hace perder el tiempo. ¿Vamos?*

Esperá confirmación corta. Si dice "no, después" → respondé la consulta original SIN bootstrap, pero NO
grabes mastery (queda `unknown` implícito y se vuelve a ofrecer la próxima).

---

## Step 2 — La misión (30 segundos, 1 pregunta)

ANTES de los probes, capturá el PARA QUÉ. Una sola pregunta:

> *Antes de medirte: ¿para qué querés esto? Decime qué stack tocás hoy en tu laburo, a qué rol apuntás y en
> qué plazo. Algunos ejemplos para orientarte:*
>
> *- "Uso Snowflake, dbt y Airflow todos los días pero no los termino de entender; quiero entenderlos bien"*
> *- "Analytics Engineer, quiero pasar a Data Platform Engineer en un año"*
> *- "Vengo de análisis de datos y me pasaron a ingeniería, estoy remando"*
> *- "Quiero estar entrevistando en 6 meses para mercado global remoto"*

De la respuesta extraé `target_role`, `current_stack`, `domain`, `market`, `timeline` y `why`. Si algún
campo no lo dice, dejalo en `general`/`sin definir` — NO interrogues de más, esto es 1 pregunta, no un
cuestionario.

> **`current_stack` es el campo más importante de esta skill.** Es lo que define qué hitos priorizar y de
> dónde salen los ejemplos de todas las lecciones. Si el usuario no lo menciona, es lo ÚNICO que vale la
> pena repreguntar.

---

## Step 3 — Background calibration (1 minuto)

Hacé esta pregunta ÚNICA con 4 opciones:

> *Segunda y última pregunta de contexto: ¿de dónde venís hoy con data engineering?*
>
> *1. **Junior** — escribo SQL y toqué alguna herramienta, no diseñé pipelines todavía*
> *2. **Mid** — mantengo pipelines y modelos que ya existían, agrego cosas, pero muchas decisiones las heredé sin entenderlas*
> *3. **Senior** — diseñé plataformas de datos, tomé decisiones de arquitectura y de costo, vengo a cubrir gaps*
> *4. **Cambio de stack** — vengo de otro mundo (análisis, backend, BI, ciencia de datos) y arranco en ingeniería de datos*

Esperá respuesta. Según el número, aplicá este **bootstrap pattern**:

### Opción 1 — Junior
- **Todos los 36 conceptos** → `unknown`
- `evidence: "bootstrap-junior-{YYYY-MM-DD}"`

### Opción 2 — Mid
- `explored`: `dimensional-modeling`, `snowflake-architecture`, `virtual-warehouses`, `dbt-project-structure`, `dbt-ref-lineage`, `materializations`, `dag-scheduling`, `operators-hooks-taskflow`, `rest-resource-design`, `azure-pipelines`
- `unknown`: el resto (26 conceptos)
- `evidence: "bootstrap-mid-{YYYY-MM-DD}"`

> Es el perfil más común del usuario objetivo de esta skill: usa las herramientas, no entiende el motor.
> Ojo con inflarlo — "lo uso" no es `explored`. Los probes del Step 4 son los que deciden.

### Opción 3 — Senior
- **Todos los 36 conceptos** → `explored`
- `evidence: "bootstrap-senior-claimed-{YYYY-MM-DD}"`
- *Nota interna*: los probes del Step 4 son ESPECIALMENTE importantes acá — si fallan, bajás varios a `unknown` ANTES de guardar.

### Opción 4 — Cambio de stack
Preguntá una sub-pregunta:
> *¿De qué venís? (análisis de datos / BI / backend / ciencia de datos / infraestructura / otro)*

Según respuesta:
- **Análisis de datos / BI**: `explored` para `dimensional-modeling`, `dbt-project-structure`, `data-serving-layer`. Resto `unknown`.
- **Backend**: `explored` para `rest-resource-design`, `api-contracts`, `api-auth`, `api-reliability`, `backend-frontend-split`. Resto `unknown`.
- **Ciencia de datos**: `explored` para `columnar-storage`, `batch-vs-streaming`, `dbt-project-structure`. Resto `unknown`.
- **Infraestructura / DevOps**: `explored` para `azure-pipelines`, `iac-secrets`, `cicd-for-data`, `airflow-architecture`, `api-auth`. Resto `unknown`.
- **Otro**: todos `unknown`.
- `evidence: "bootstrap-stackswitch-{origen}-{YYYY-MM-DD}"`

---

## Step 4 — Conceptual probes (2 minutos, 3 preguntas)

Independientemente de la opción anterior, hacé estos 3 probes para refinar. Tono Gentleman, sin embromar.
Los tres están elegidos porque son exactamente donde vive la caja negra del usuario objetivo.

### Probe 1 — `micro-partitions` (el corazón de Snowflake)

> *En Snowflake no hay índices. Si filtro una tabla de mil millones de filas por una columna, ¿cómo hace
> para no leerla entera? Contámelo con tus palabras.*

**Lectura de respuesta**:
- **Buena** (menciona que los datos están en bloques con metadata de rangos, y que descarta bloques enteros sin leerlos): mantené `micro-partitions` en el nivel del bootstrap, o subilo a `explored` si estaba `unknown`.
- **Media** ("está optimizado", "usa estadísticas", sin la idea de descartar bloques): dejá en `unknown` y anotá que la idea de pruning no está formada.
- **Mala / "no sé" / "creo que igual escanea todo"**: forzá `micro-partitions` a `unknown` y registrá misconception si dijo algo específico y errado (ej. "tiene índices ocultos").

### Probe 2 — `dag-scheduling` + `dag-idempotency` (el corazón de Airflow)

> *Tenés un DAG que corre todos los días a las 3 de la mañana. ¿Qué datos procesa la corrida de hoy? ¿Y qué
> pasa si lo reejecutás dentro de tres meses?*

**Lectura**:
- **Buena** (dice que procesa la ventana anterior, y que reejecutar debería dar el mismo resultado si usa la fecha del intervalo): mantené ambos conceptos en nivel de bootstrap.
- **Media** (acierta la ventana pero no ve el problema del reproceso, o al revés): bajá `dag-idempotency` a `unknown`, dejá `dag-scheduling` como estaba.
- **Mala** ("procesa los datos de hoy" + "corre igual"): bajá los dos a `unknown`. Es la misconception más común del hito 4 — registrala explícitamente.

### Probe 3 — `materializations` + `incremental-models` (el corazón de dbt)

> *¿Cuándo conviene que un modelo dbt sea incremental en vez de tabla? ¿Y qué puede salir mal con un
> incremental?*

**Lectura**:
- **Buena** (relaciona con costo de reconstrucción vs volumen que cambia, Y menciona algún riesgo real como datos que llegan tarde o divergencia con full-refresh): mantené ambos.
- **Media** (dice "cuando la tabla es grande" sin razón de costo, y no ve riesgos): dejá `materializations` como estaba, bajá `incremental-models` a `unknown`.
- **Mala** ("incremental es siempre mejor" / "no sé qué puede salir mal"): bajá los dos a `unknown` y registrá la misconception "cree que incremental es siempre mejor".

---

## Step 5 — Dirección (30 segundos)

Después de los probes:

> *Listo, te calibré. Tenés mastery state guardado, así que de ahora en adelante no te vuelvo a preguntar
> nada de esto — me ajusto solo. ¿Por dónde arrancás?*
>
> *a) **El concepto donde te trabaste en el probe** (te lo destrabo ya)*
> *b) **El hito de la herramienta que más usás** en tu laburo*
> *c) **Tu pregunta original**, que veníamos a responder antes del bootstrap*
> *d) **Hito 1** desde los fundamentos (recomendado si querés la base sólida antes de las herramientas)*

Esperá respuesta y proceder.

---

## Step 6 — Guardar bootstrap en engram (CRÍTICO, no saltar)

### 6a — La misión

```yaml
topic_key: skill/data-engineer-mentor/mission
type: preference
title: "Misión del usuario: {target_role}"
content: |
  target_role: "{ej. Analytics Engineer, Data Platform Engineer, Data Architect}"
  current_stack: "{ej. Snowflake + dbt + Airflow + Azure DevOps}"
  domain: "{ej. retail, fintech, salud, general}"
  market: "{es-AR | global | latam | mixed}"
  timeline: "{ej. entender mejor lo que uso | 6 meses para entrevistar | sin definir}"
  why: "{1-2 oraciones — la razón real}"
  updated: "{YYYY-MM-DD}"
```

### 6b — Mastery por concepto

Para cada uno de los 36 conceptos de `concepts.md`:

```yaml
topic_key: skill/data-engineer-mentor/mastery/{concept-slug}
type: pattern
title: "Mastery bootstrap: {concept-slug} = {level}"
content: |
  **What**: Bootstrap inicial del concepto `{concept-slug}` en nivel `{level}`.
  **Why**: Onboarding de la skill senior-data-engineer-mentor, opción {1-4}.
  **Where**: skill engram namespace.
  **Learned**: {nota del probe si aplica}
  ---
  level: {unknown | explored | practiced | mastered}
  evidence: "bootstrap-{opcion}-{YYYY-MM-DD}"
  last_validated: "{YYYY-MM-DD}"
  next_review: "{last_validated + intervalo base del nivel; unknown → sin fecha}"
  ease: 2.0
  misconceptions: ["{si un probe reveló una creencia errónea específica + fecha}"]
  history:
    - "{YYYY-MM-DD}: bootstrap from option {N}"
```

**Optimización**: si el bootstrap es 100% `unknown` (opción 1 sin variaciones de probe), guardá UNA
observación resumen en `skill/data-engineer-mentor/mastery/_bootstrap-snapshot` con la lista completa, y
creá entradas individuales a medida que cada concepto se trabaja. La skill lee el snapshot como fallback.

**Índice** (para que futuras búsquedas lo encuentren rápido):

```yaml
topic_key: skill/data-engineer-mentor/prefs/bootstrap-done
title: "Bootstrap completado"
content: |
  **What**: Onboarding 4-min completado, opción {N}.
  **When**: {YYYY-MM-DD}
  **Concepts initialized**: 36
```

---

## Edge cases

- **Usuario dice "no me hagas el bootstrap, andá directo"** → respondé la consulta original, NO guardes mastery, ofrecé hacerlo después.
- **Usuario responde con bronca o evasivo a los probes** → cortá los probes y bootstrappeá conservador (un nivel menos que el background declarado).
- **Usuario dice "uso todo eso hace años" en la opción 3 pero falla los 3 probes** → NO discutas. Bajá lo que corresponda y decilo con respeto: *"Che, usarlo y entender el motor son cosas distintas — y justamente para eso está esta skill. Te dejo esos tres en `unknown` y arrancamos por ahí."*
- **Usuario salta de opción a opción** → quedate con la última y procedé.
- **Si engram falla (`mem_save` da error)** → avisá: *"No pude persistir tu mastery state, voy a tener que repreguntarte la próxima. Igual seguimos."*
