# Diagnostic Probes — uno por concepto (36)

**Cuándo se carga**: cuando hay que decidir si un concepto sube de `unknown` a `explored`, o cuando toca un
repaso vencido (`next_review <= hoy`).

**Regla dura de uso**: el probe se contesta **a libro cerrado**. Si el usuario tiene el material a la vista,
o si acabás de explicárselo en este mismo turno, el resultado NO cuenta para subir de nivel — eso es
fluidez, no retención. Un probe válido se hace en otro momento, sobre algo que ya se enseñó antes.

**Regla de diseño** (de `teach`): si alguna vez convertís un probe a opción múltiple, **todas las opciones
llevan la misma cantidad de palabras**. La correcta no puede ser la más larga ni la mejor redactada.

**Cómo leer las respuestas**: cada probe trae tres lecturas.
- **Buena** → el concepto sube a `explored` (o se mantiene si ya estaba más arriba) y se recalcula `next_review` con `ease` aumentado.
- **Media** → se mantiene el nivel actual, sin subir. Si venía de un repaso, se resetea el intervalo al base.
- **Mala** → queda o vuelve a `unknown`, y si la respuesta contiene una creencia ERRÓNEA específica (no solo desconocimiento), se appendea a `misconceptions[]` textualmente.

---

## Hito 1 — Fundamentos

### `de-lifecycle`
> *Nombrame las etapas por las que pasa un dato desde que se genera hasta que alguien lo consume, y decime en cuál de ellas trabajás vos.*

- **Buena**: nombra generación, ingesta, almacenamiento, transformación y serving (aunque use otras palabras), y ubica su trabajo entendiendo que hay etapas antes y después con dueños distintos.
- **Media**: nombra el medio ("traigo, transformo, cargo") sin ver lo de antes ni lo de después.
- **Mala**: describe solo su tarea diaria sin noción de cadena.

### `dimensional-modeling`
> *¿Qué es el grano de una tabla de hechos y por qué importa antes de escribir el SQL?*

- **Buena**: dice que es qué representa una fila, y conecta con que define qué métricas son sumables y evita duplicaciones.
- **Media**: dice "el nivel de detalle" sin conectar con la consecuencia.
- **Mala**: no lo distingue de la granularidad de la fuente, o confunde con particionamiento. *Misconception frecuente: "el grano es lo mismo que la clave primaria".*

### `columnar-storage`
> *¿Por qué `SELECT *` sobre una tabla ancha cuesta mucho más que pedir tres columnas, en un warehouse analítico?*

- **Buena**: explica que el almacenamiento es por columna, así que solo se leen del disco las columnas pedidas.
- **Media**: dice "porque trae más datos" sin la idea de lectura selectiva por columna.
- **Mala**: cree que da igual porque "la tabla se lee entera igual".

### `batch-vs-streaming`
> *El negocio te pide "tiempo real". ¿Qué le preguntás antes de diseñar nada?*

- **Buena**: pregunta qué decisión se toma con el dato y en qué ventana; conecta la respuesta con el costo.
- **Media**: pregunta "cada cuánto lo necesitan" sin conectar con el costo ni con la acción.
- **Mala**: arranca a proponer Kafka.

### `idempotency-backfill`
> *Corrés dos veces el mismo pipeline sobre la misma ventana. ¿Qué tiene que pasar, y qué lo garantiza?*

- **Buena**: dice que el resultado tiene que ser idéntico, y menciona ventana como parámetro externo y escritura por merge o reemplazo de partición.
- **Media**: dice "no se tiene que duplicar" sin saber qué lo garantiza.
- **Mala**: cree que Airflow o dbt lo resuelven solos. *Misconception frecuente: "si uso `MERGE` ya es idempotente".*

### `table-formats`
> *¿Qué te da Iceberg o Delta que no te da tener Parquets sueltos en un bucket?*

- **Buena**: menciona metadata que define qué archivos componen la tabla, y de ahí commits atómicos, time travel y borrado por fila.
- **Media**: dice "versionado" o "time travel" sin entender de dónde sale.
- **Mala**: cree que es un formato de archivo distinto a Parquet.

---

## Hito 2 — Snowflake

### `snowflake-architecture`
> *¿Por qué en Snowflake podés apagar el compute sin perder los datos, y por qué dos equipos no se pisan?*

- **Buena**: explica que el storage está separado del compute y es central, y que cada warehouse es un cluster independiente sobre los mismos datos.
- **Media**: dice "porque está en la nube" sin la idea de separación de capas.
- **Mala**: cree que cada warehouse tiene su copia de los datos.

### `micro-partitions`
> *En Snowflake no hay índices. ¿Cómo evita leer una tabla entera al filtrar?*

- **Buena**: bloques con metadata de rangos por columna, se descartan bloques enteros sin leerlos.
- **Media**: "usa estadísticas" sin la idea de descarte de bloques.
- **Mala**: cree que igual escanea todo, o que hay índices ocultos. *Misconception frecuente: "la clustering key es un índice".*

### `virtual-warehouses`
> *Tus queries están todas encoladas pero cada una corre rápido. ¿Agrandás el warehouse o le agregás clusters? ¿Por qué?*

- **Buena**: agrega clusters, porque el problema es concurrencia y no recursos por query.
- **Media**: elige bien pero no puede explicar por qué.
- **Mala**: agranda el warehouse. *Misconception frecuente: "más grande siempre es más rápido".*

### `snowflake-caching`
> *Corrés una query, tarda 40 segundos. La corrés de nuevo y tarda 1 segundo. ¿Optimizaste algo?*

- **Buena**: no, actuó el result cache; y para medir de verdad hay que mirar bytes escaneados o particiones podadas.
- **Media**: identifica el cache pero no sabe cómo medir sin él.
- **Mala**: cree que la query mejoró.

### `clustering-pruning`
> *¿Cuándo NO pondrías una clustering key en una tabla?*

- **Buena**: en tablas no muy grandes, en tablas que se recrean enteras, con columnas de altísima cardinalidad, o cuando el costo de reclustering supera el ahorro.
- **Media**: dice "cuando no hace falta" sin criterio.
- **Mala**: cree que siempre conviene.

### `snowflake-cost`
> *Te dicen que la factura de Snowflake subió 40%. ¿Cuál es tu primer paso?*

- **Buena**: identificar QUÉ warehouse creció comparando el consumo mes contra mes, antes de tocar nada.
- **Media**: propone optimizar queries directamente, sin ubicar el origen.
- **Mala**: propone achicar warehouses a ciegas.

---

## Hito 3 — dbt

### `dbt-project-structure`
> *¿Qué va y qué NO va en la capa de staging?*

- **Buena**: renombrar, castear, limpiar tipos; una fuente por modelo; NADA de lógica de negocio.
- **Media**: describe staging como "la primera capa" sin la regla de responsabilidad única.
- **Mala**: mete lógica de negocio ahí "para no repetirla".

### `dbt-ref-lineage`
> *¿Qué se rompe si escribís `FROM analytics.dim_cliente` en vez de `ref('dim_cliente')`?*

- **Buena**: dbt pierde la dependencia, con lo cual el orden de ejecución, el lineage y la separación de ambientes dejan de funcionar.
- **Media**: dice "no queda en el lineage" sin ver el problema de orden ni de ambientes.
- **Mala**: cree que es equivalente.

### `materializations`
> *¿Con qué criterio elegís entre vista, tabla e incremental?*

- **Buena**: compara costo de reconstrucción contra frecuencia de consulta, y menciona que incremental agrega complejidad que hay que justificar.
- **Media**: dice "tabla si es lento consultarla" sin el eje de costo de build.
- **Mala**: usa un default fijo para todo.

### `incremental-models`
> *Nombrame lo que tenés que decidir sí o sí al hacer un modelo incremental.*

- **Buena**: clave única, ventana de lookback para datos que llegan tarde, qué hacer con updates y deletes, y cada cuánto se hace full-refresh de reconciliación.
- **Media**: menciona clave única y nada más.
- **Mala**: cree que alcanza con filtrar por fecha. *Misconception frecuente: "filtrar contra `MAX()` del destino es suficiente".*

### `dbt-tests-contracts`
> *Si solo pudieras poner UN test en un mart nuevo, ¿cuál ponés y por qué?*

- **Buena**: unicidad sobre la clave del grano, porque su violación duplica métricas y es el error más caro y más difícil de ver.
- **Media**: elige `not_null` o freshness con justificación razonable pero sin la jerarquía de impacto.
- **Mala**: no puede priorizar, o dice "todos".

### `dbt-jinja-macros`
> *¿En qué momento se ejecuta el Jinja de un modelo dbt, y por qué eso importa?*

- **Buena**: antes de que exista el SQL, en compilación; por eso Jinja no puede tomar decisiones basadas en los datos del propio modelo.
- **Media**: sabe que "compila primero" pero no deriva la consecuencia.
- **Mala**: cree que corre junto con el SQL en el warehouse.

---

## Hito 4 — Airflow

### `airflow-architecture`
> *Un DAG figura como corriendo pero ninguna tarea arranca. ¿Qué revisás primero?*

- **Buena**: disponibilidad de slots (pools, límites de concurrencia), después workers vivos, scheduler sano y metadata DB.
- **Media**: revisa logs sin un orden de hipótesis.
- **Mala**: reinicia todo.

### `dag-scheduling`
> *Un DAG diario corre a las 3 AM. ¿Qué ventana de datos procesa esa corrida?*

- **Buena**: la ventana anterior — la corrida se dispara al cerrar el intervalo.
- **Media**: duda pero sabe que "no es exactamente hoy".
- **Mala**: dice "los datos de hoy". *Misconception frecuente y la más importante del hito: "Airflow es un cron".*

### `operators-hooks-taskflow`
> *¿Qué pasa si pasás un DataFrame grande por XCom?*

- **Buena**: se guarda en la metadata database, la hincha, la vuelve lenta para toda la instancia y eventualmente falla.
- **Media**: sabe que "no conviene" sin saber por qué.
- **Mala**: no ve problema.

### `dag-idempotency`
> *¿Por qué `datetime.now()` adentro de una tarea es un bug aunque el DAG corra puntual todos los días?*

- **Buena**: porque en un reintento o un backfill devuelve el momento real de ejecución y no la ventana que corresponde, produciendo datos incorrectos sin dar error.
- **Media**: dice "porque no es reproducible" sin el caso concreto.
- **Mala**: no ve el problema mientras corra a horario.

### `airflow-assets`
> *Dos DAGs encadenados: el segundo arranca 30 minutos después del primero. ¿Qué puede salir mal?*

- **Buena**: si el primero se atrasa, el segundo corre con datos incompletos y no falla — falla silenciosa.
- **Media**: dice "puede no estar listo" sin señalar que no hay error.
- **Mala**: no ve riesgo porque "siempre alcanza".

### `airflow-scaling`
> *Agregaste workers y Airflow sigue lento. ¿Qué puede estar pasando?*

- **Buena**: hay tareas ocupando slots sin trabajar (sensores esperando), o el cuello está en el scheduler o en la metadata DB, no en capacidad de cómputo.
- **Media**: sospecha de configuración sin ubicar dónde.
- **Mala**: propone agregar más workers.

---

## Hito 5 — APIs & MCP

### `rest-resource-design`
> *¿Por qué está mal devolver 200 con un campo `error` adentro del cuerpo?*

- **Buena**: porque toda la infraestructura intermedia (proxies, reintentadores, monitoreo) lee el código de estado; con 200 el error es invisible para todos menos para ese cliente.
- **Media**: dice "no es correcto" sin las consecuencias.
- **Mala**: no ve problema si el cliente lo entiende.

### `api-contracts`
> *¿Qué cambio en una API rompe a los consumidores y cuál no?*

- **Buena**: agregar campos opcionales normalmente no rompe; sacar campos, cambiar tipos o volver obligatorio lo opcional sí. Menciona además el caso invisible: cambiar el significado sin cambiar el tipo.
- **Media**: da la regla básica sin los casos grises.
- **Mala**: cree que cualquier cambio rompe, o que ninguno rompe.

### `api-pagination-filtering`
> *Paginás por offset una API mientras la fuente recibe inserts. ¿Qué pasa?*

- **Buena**: las posiciones se desplazan entre páginas, así que duplicás o salteás registros — y el salteo no deja rastro.
- **Media**: sabe que "puede haber problemas" sin explicar el mecanismo.
- **Mala**: no ve riesgo.

### `api-auth`
> *Tu job dura 3 horas y el token expira cada hora. ¿Cómo lo resolvés?*

- **Buena**: renovar el token durante la ejecución, al vencer o ante un 401, en vez de pedirlo una sola vez al inicio.
- **Media**: propone alargar el token o partir el job.
- **Mala**: no había pensado en el vencimiento.

### `api-reliability`
> *¿Qué errores reintentás y cuáles no?*

- **Buena**: transitorios (timeout, 429, 5xx) con retroceso y jitter; permanentes (4xx salvo 429) no se reintentan porque no cambian.
- **Media**: distingue algo pero sin la categoría de "no reintentar".
- **Mala**: reintenta todo "por las dudas".

### `mcp-protocol`
> *¿Qué diferencia hay entre una tool y un resource en MCP, y por qué importa esa distinción?*

- **Buena**: tool es acción con efecto o costo, resource es dato leíble como contexto; importa porque tienen modelos de permiso y de riesgo distintos.
- **Media**: sabe la definición sin la consecuencia de permisos.
- **Mala**: cree que MCP es lo mismo que llamar funciones desde un LLM. *Misconception frecuente.*

---

## Hito 6 — System Design & Delivery

### `backend-frontend-split`
> *El equipo de producto quiere que el dashboard consulte Snowflake directo. ¿Qué respondés?*

- **Buena**: nombra al menos dos de los cuatro problemas (credenciales expuestas, autorización por fila, latencia, costo por usuario) Y propone la alternativa concreta.
- **Media**: dice que está mal sin argumentos o sin alternativa.
- **Mala**: no ve problema.

### `data-serving-layer`
> *¿Por qué el warehouse no es un buen servidor de aplicaciones?*

- **Buena**: está optimizado para escanear mucho con poca concurrencia, no para responder muchas consultas chicas con baja latencia; y el costo escala con el uso de la app.
- **Media**: dice "es lento" sin el eje de concurrencia ni de costo.
- **Mala**: cree que da igual si la query es rápida.

### `azure-pipelines`
> *Una variable te aparece vacía en un paso del pipeline. ¿Cuál es tu primera hipótesis?*

- **Buena**: que se está evaluando en un momento distinto al que se define — expresión de compilación para algo que solo existe en ejecución, o sustitución de una variable inexistente.
- **Media**: revisa la definición sin pensar en el momento de evaluación.
- **Mala**: prueba sintaxis al azar.

### `cicd-for-data`
> *¿Por qué no correr todo el proyecto dbt en cada pull request?*

- **Buena**: reconstruir todo cuesta compute y tiempo desproporcionados; con el DAG se puede construir solo lo modificado y sus descendientes y diferir el resto a producción.
- **Media**: dice "es lento" sin conocer la alternativa.
- **Mala**: cree que hay que correr todo para estar seguro.

### `iac-secrets`
> *Commiteaste una credencial y la borraste en el commit siguiente. ¿Estás cubierto?*

- **Buena**: no — queda en el historial y en todos los clones; hay que rotar el secreto.
- **Media**: duda pero intuye que no alcanza.
- **Mala**: cree que borrarla resuelve.

### `data-governance-cost`
> *Vas a cambiar el grano de un mart. ¿Cómo sabés quién se rompe?*

- **Buena**: lineage para los consumidores internos MÁS el log de consultas del warehouse para los externos (dashboards, notebooks) que el lineage no ve.
- **Media**: solo menciona el lineage.
- **Mala**: cree que basta con avisar en un canal.

---

## Protocolo de aplicación

1. Elegí el probe del concepto que estás por enseñar o repasar.
2. Hacé la pregunta **sola**, sin contexto que la responda. Esperá.
3. NO des pistas mientras el usuario piensa.
4. Clasificá la respuesta con la rúbrica de arriba.
5. Actualizá `mem_save` sobre `skill/data-engineer-mentor/mastery/{slug}`: nivel, `evidence`, `last_validated`, `next_review`, `ease`, y `misconceptions[]` si apareció una creencia errónea específica.
6. Si fue **mala** y el concepto estaba en `practiced` o `mastered`, disparás la regla de degradación con aviso explícito y derecho a veto.
7. Si hay 2-3 conceptos vencidos del mismo trío de interleaving (ver final de `concepts.md`), aplicá los probes **alternados**, no uno tras otro.
