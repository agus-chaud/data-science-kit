# Hito 1 — Fundamentos de Data Engineering

## Por qué importa (perspectiva corporativa)

Hermano, te lo digo directo: la mayoría de la gente que "hace data engineering" aprendió las herramientas
sin aprender el oficio. Saben escribir un modelo dbt pero no saben qué es el grano de una tabla de hechos.
Saben correr un DAG pero no saben qué significa que un pipeline sea idempotente. Y eso se nota EXACTAMENTE
en el momento en que algo se rompe: cuando hay que reprocesar tres meses de datos, cuando el mart empieza a
duplicar filas, cuando el costo de Snowflake se triplica sin que nadie sepa por qué.

Este hito es el que separa al que **opera** herramientas del que **diseña** sistemas de datos. Y en el
mercado eso es la diferencia entre un Analytics Engineer junior y un Data Platform Engineer senior — que en
Argentina son dos rangos salariales completamente distintos.

Acá está el detalle que la mayoría no entiende: **estos conceptos son anteriores a las herramientas y les
sobreviven**. Kimball escribió sobre modelado dimensional en los 90 y hoy tu proyecto dbt lo implementa sin
que nadie lo nombre. El formato columnar es de 2010 y es la razón por la que Snowflake puede hacer lo que
hace. Idempotencia es un concepto de sistemas distribuidos y es lo que hace que un backfill de Airflow no
te destruya la tabla. Si entendés esta capa, las herramientas se vuelven obvias. Si no, cada herramienta es
una caja negra distinta que hay que memorizar.

## Conceptos de este hito

### de-lifecycle

**Qué es**: El ciclo de vida del dato: **generación → ingesta → almacenamiento → transformación → serving**,
atravesado por corrientes de fondo (*undercurrents*) que aplican a TODAS las etapas: seguridad, gestión de
datos, DataOps, arquitectura, orquestación e ingeniería de software.

**La trampa del junior**: pensar que su trabajo empieza cuando el dato llega al lake y termina cuando el
modelo dbt corre verde. Todo lo de antes ("eso es del equipo de la app") y todo lo de después ("eso lo ve
el analista") queda fuera de su cabeza. Resultado: pipelines que se rompen por cambios de esquema que nadie
avisó, y marts que nadie usa porque no responden la pregunta del negocio.

**Cómo lo piensa un senior**: el dato tiene **dueños en cada etapa** y el trabajo del data engineer es
gestionar los CONTRATOS entre etapas, no solo el código del medio. La pregunta de un senior nunca es "¿cómo
transformo esto?" — es "¿quién produce este dato, con qué garantía, y quién lo consume esperando qué?".
Las undercurrents no son opcionales: un pipeline sin observabilidad ni seguridad no es un pipeline, es una
deuda.

**Tradeoffs reales**:

| Decisión de arquitectura | Cuándo conviene | Costo |
|---|---|---|
| ETL (transformo antes de cargar) | Compliance estricto, PII que no puede tocar el warehouse | Menos flexible, reprocesar es caro |
| ELT (cargo crudo, transformo adentro) | Default moderno con warehouse elástico (Snowflake) | Almacenás basura, costo de compute adentro |
| Lake → Warehouse (dos capas) | Datos semiestructurados voluminosos + analítica | Dos sistemas que mantener y sincronizar |
| Lakehouse (una capa con table format) | Querés una sola copia con ACID y motores múltiples | Madurez operativa mayor, tooling más nuevo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Diferencia entre ETL y ELT y por qué ELT ganó?*
  A: En ETL transformás en un motor intermedio antes de cargar; en ELT cargás crudo y transformás dentro del warehouse. ELT ganó porque el compute del warehouse se volvió elástico y barato: es más rápido cargar todo y transformar con SQL, y guardás el crudo para reprocesar sin volver a la fuente. ETL sigue vigente cuando hay restricciones de compliance o cuando la fuente no puede exportar en volumen.
- Q (senior): *Un dato llega mal al mart. ¿Cómo ubicás en qué etapa se rompió?*
  A: De atrás para adelante, con evidencia en cada corte: comparo el mart contra la capa intermedia, la intermedia contra staging, staging contra el raw ingerido, y el raw contra la fuente. El corte donde el número cambia es la etapa culpable. Sin esa disciplina, la gente "arregla" el mart y el bug vuelve la semana siguiente.
- Q (trampa): *¿El data engineer es responsable de la calidad del dato de la fuente?*
  A: Responsable de la calidad no, responsable de DETECTARLA sí. No podés controlar que la app mande un campo nulo, pero sí tenés que tener un test que lo cace antes de que llegue al mart y un contrato con el equipo dueño. "Es culpa de la fuente" no es una respuesta aceptable en producción.

### dimensional-modeling

**Qué es**: El modelo de Kimball: tablas de **hechos** (eventos medibles, métricas numéricas, claves foráneas)
rodeadas de tablas de **dimensiones** (el contexto: quién, qué, cuándo, dónde) en **star schema**. El
**grano** define qué representa exactamente una fila de la tabla de hechos.

**La trampa del junior**: armar el mart "como salió", mezclando granos distintos en la misma tabla (una fila
por línea de pedido y otra por pedido completo), o metiendo métricas en las dimensiones. Después las sumas
dan mal y nadie entiende por qué, y aparece el clásico "el dashboard no cierra con el sistema".

**Cómo lo piensa un senior**: la PRIMERA decisión, antes de escribir una línea de SQL, es declarar el
**grano en una oración**: *"una fila = una línea de factura por producto por día"*. Todo lo demás se deriva.
Si no podés escribir esa oración, no entendiste el requerimiento todavía. Las dimensiones se desnormalizan a
propósito (no es "mal diseño", es diseño analítico), y los cambios históricos se manejan con **SCD Type 2**
cuando el negocio necesita ver el pasado como era, no como es hoy.

**Tradeoffs reales**:

| Patrón | Cuándo conviene | Contra |
|---|---|---|
| Star schema (Kimball) | Default para marts de consumo BI | Duplicación en dimensiones, más modelos |
| Snowflake schema (normalizado) | Dimensiones enormes con jerarquías repetidas | Más joins, peor performance, peor UX de BI |
| One Big Table (OBT) | Consumo por una sola herramienta, warehouse columnar | Explota si hay muchos consumidores distintos, difícil de mantener |
| Data Vault | Trazabilidad y auditoría extremas, muchas fuentes | Complejidad alta, no es capa de consumo |
| SCD Type 1 (sobrescribe) | El histórico no importa (corrección de typos) | Perdés el pasado para siempre |
| SCD Type 2 (versiona filas) | El negocio necesita "cómo era en ese momento" | Más filas, joins con rango de fechas, más difícil de consultar |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es el grano de una tabla de hechos?*
  A: Qué representa exactamente una fila. Es la primera decisión del diseño y define qué métricas son sumables y a qué nivel. Mezclar granos en una tabla es el error que hace que las sumas se dupliquen.
- Q (senior): *El negocio pide ver las ventas por vendedor "como era el vendedor en ese momento", pero los vendedores cambian de zona. ¿Qué hacés?*
  A: SCD Type 2 en la dimensión de vendedor: cada cambio de zona genera una fila nueva con `valid_from` / `valid_to` y un flag `is_current`. La tabla de hechos referencia la clave surrogate vigente al momento del evento, no el id natural. Si referenciaras el id natural, al cambiar la zona se reescribiría el pasado.
- Q (trampa): *Con warehouses columnares y compute barato, ¿el modelado dimensional quedó obsoleto?*
  A: No. La performance dejó de ser la razón principal, pero las otras dos siguen: comprensibilidad para el consumidor de negocio y consistencia de métricas entre reportes. Una OBT sin grano declarado te da los mismos números inconsistentes que antes, solo que más rápido. El modelado es un contrato semántico, no una optimización.

### columnar-storage

**Qué es**: Guardar los datos **por columna** en vez de por fila. Habilita compresión mucho mayor (los
valores de una columna son homogéneos), *encoding* especializado (dictionary, run-length), y sobre todo
**projection pushdown** (leo solo las columnas que pido) y **predicate pushdown** (salteo bloques enteros
usando estadísticas).

**La trampa del junior**: escribir `SELECT *` por costumbre y después no entender por qué la misma consulta
sobre la misma tabla cuesta 10 veces más que la del compañero. O guardar millones de archivitos Parquet de
1 MB, matando exactamente la ventaja que el formato le daba.

**Cómo lo piensa un senior**: columnar es **la razón física** por la que Snowflake, BigQuery, Redshift y
Parquet existen. En analítica leés pocas columnas de muchísimas filas; en OLTP leés muchas columnas de pocas
filas. Ese solo hecho explica por qué no usás Postgres para el warehouse ni Snowflake para el checkout de
la app. Y explica el otro gran costo: **archivos chicos son veneno** — cada archivo tiene overhead de
metadata y de apertura, así que 10.000 archivos de 1 MB rinden peor que 100 de 100 MB.

**Tradeoffs reales**:

| Formato | Cuándo | Contra |
|---|---|---|
| Parquet | Default analítico en lake, columnar + comprimido | No es appendable eficiente, sin ACID por sí solo |
| ORC | Ecosistema Hive/Presto establecido | Menos soporte fuera de ese mundo |
| Avro | Ingesta row-oriented, schema evolution, streaming | Mala performance analítica (es por fila) |
| CSV / JSON | Interoperabilidad, debug humano | Sin tipos, sin compresión, escaneo completo siempre |
| Snowflake interno (micro-partitions) | Dentro de Snowflake, gestionado automáticamente | Formato propietario, no lo leés desde afuera |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué columnar es mejor para analítica?*
  A: Porque las queries analíticas tocan pocas columnas sobre muchas filas. Columnar lee solo esas columnas del disco (projection pushdown) y comprime mucho mejor porque los valores de una columna son del mismo tipo y suelen repetirse. En row-oriented tenés que leer la fila entera aunque uses dos campos.
- Q (senior): *¿Por qué "many small files" es un problema si el total de bytes es el mismo?*
  A: Porque el costo no es solo de bytes: cada archivo implica una operación de listado y apertura, y su metadata/footer se lee entero. Con miles de archivos chicos el tiempo se va en overhead de I/O y en el planner, no en leer datos. Además la compresión rinde peor porque los bloques son chicos. La solución es compactación periódica.
- Q (trampa): *¿Comprimir siempre conviene?*
  A: Casi siempre en analítica, pero hay tradeoff: más compresión = menos I/O pero más CPU para descomprimir. Con codecs modernos (Snappy, ZSTD) el balance favorece comprimir, porque el I/O suele ser el cuello de botella. Si el cuello es CPU y el storage es local y rápido, la ecuación cambia.

### batch-vs-streaming

**Qué es**: Batch procesa conjuntos acotados en intervalos; streaming procesa eventos a medida que llegan;
en el medio está el **micro-batch** (intervalos cortos) y el **CDC** (Change Data Capture, capturar cambios
de la base fuente desde su log de transacciones).

**La trampa del junior**: pedir streaming porque "el negocio quiere tiempo real", sin preguntar qué decisión
se toma con ese dato ni cada cuánto. Terminan con una infraestructura 5 veces más cara y frágil para
alimentar un dashboard que alguien mira dos veces por día.

**Cómo lo piensa un senior**: la pregunta correcta NUNCA es "¿batch o streaming?" — es **"¿cuál es el costo
de que este dato tenga N minutos de atraso?"**. Si nadie puede cuantificar ese costo, la respuesta es batch.
Streaming se justifica cuando hay una acción automática con ventana corta: detección de fraude, alertas
operativas, personalización en sesión. Para reporting, micro-batch cada 15 minutos cubre el 95% de los
casos a una fracción del costo y de la complejidad operativa.

**Tradeoffs reales**:

| Approach | Latencia | Costo / complejidad | Cuándo |
|---|---|---|---|
| Batch diario | horas | Mínimo | Reporting, marts, cierres contables |
| Micro-batch (15-60 min) | minutos | Bajo | Dashboards operativos — cubre casi todo |
| CDC a warehouse | minutos | Medio (conector + monitoreo) | Réplica de operacional sin castigar la fuente |
| Streaming real (Kafka + procesador) | segundos | Alto (infra + skills + on-call) | Fraude, alertas, personalización en vivo |
| Snowpipe / auto-ingest | minutos | Bajo-medio | Archivos que caen a storage continuamente |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es CDC y qué problema resuelve?*
  A: Capturar los cambios de una base fuente leyendo su log de transacciones en vez de consultarla. Resuelve dos cosas: no castigás la base operacional con queries pesadas, y capturás deletes y updates que un `WHERE updated_at > x` te perdería.
- Q (senior): *El negocio pide "tiempo real". ¿Cómo lo aterrizás?*
  A: Le pregunto qué decisión toma con ese dato y en qué ventana. Si la respuesta es "miro el dashboard a la mañana", la latencia tolerable es de horas y batch alcanza. Si es "bloqueo la transacción sospechosa", ahí sí hay caso de streaming. Después pongo el número de costo de cada opción al lado del requerimiento — casi siempre el requerimiento se recalibra solo.
- Q (trampa): *¿Streaming es siempre más caro que batch?*
  A: En infraestructura y en costo humano casi siempre sí. Pero hay un caso donde no: cuando el batch te obliga a reprocesar ventanas grandes repetidamente. Si tu batch horario recomputa 30 días cada vez, el streaming incremental puede salir más barato en compute. Hay que medir, no asumir.

### idempotency-backfill

**Qué es**: Que correr el mismo proceso N veces con el mismo input produzca el mismo resultado que correrlo
una vez. Es lo que permite reintentar, reprocesar y hacer **backfill** (recomputar el pasado) sin miedo.

**La trampa del junior**: pipelines con `INSERT INTO` puro. Se cae en el medio, lo reintenta, y ahora tiene
filas duplicadas. O usa `CURRENT_DATE` adentro de la transformación, así que reprocesar ayer con el código de
hoy produce datos distintos — y encima nadie se da cuenta hasta el cierre del mes.

**Cómo lo piensa un senior**: **la fecha del proceso viene de afuera, nunca de adentro**. El pipeline recibe
la ventana a procesar como parámetro (`data_interval_start` en Airflow, `var()` en dbt) y escribe de forma
que reescribir esa ventana sea seguro: `MERGE` con clave única, o borrar-e-insertar la partición completa.
Un pipeline no idempotente es un pipeline que no se puede operar: no podés reintentarlo, no podés
reprocesarlo, y cada incidente se convierte en una cirugía manual a las 3 de la mañana.

**Tradeoffs reales**:

| Estrategia de escritura | Idempotente | Cuándo | Contra |
|---|---|---|---|
| `INSERT` puro | ❌ | Nunca en producción sin dedupe posterior | Duplica en cada reintento |
| `MERGE` / upsert por clave | ✅ | Default cuando hay clave natural confiable | Costoso en tablas grandes, necesita clave real |
| Delete + insert de partición | ✅ | Datos particionados por fecha | Ventana de inconsistencia si no es transaccional |
| `INSERT OVERWRITE` de partición | ✅ | Lake / tablas particionadas | Requiere que la partición sea la unidad natural |
| Append + dedupe en la lectura | ✅ (efectivamente) | Eventos inmutables de alto volumen | Costo movido a cada consulta |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué hace idempotente a un pipeline?*
  A: Que correrlo dos veces con el mismo input deje el destino igual que corriéndolo una vez. En la práctica: la ventana temporal viene como parámetro externo, y la escritura es un merge por clave o un reemplazo completo de la partición, nunca un insert ciego.
- Q (senior): *Tenés que reprocesar 6 meses de un mart. ¿Cómo lo encarás?*
  A: Primero verifico que el pipeline sea idempotente y que la fuente cruda de esos 6 meses todavía exista (si no, el backfill es imposible y esa es la conversación real). Después reproceso por particiones, no todo junto: chunks por mes o por día, con control de concurrencia para no reventar el warehouse, validando conteos por chunk contra la fuente. Y lo corro contra un esquema paralelo antes de tocar producción si el mart tiene consumidores activos.
- Q (trampa): *Tu pipeline usa `MERGE` con `unique_key`. ¿Ya es idempotente?*
  A: No necesariamente. Si el `unique_key` no es realmente único en el origen, el merge puede fallar o actualizar filas arbitrarias. Y si la transformación usa una función no determinística (`CURRENT_TIMESTAMP`, un random, o un `ROW_NUMBER()` sin orden estable), el resultado cambia entre corridas aunque el merge esté bien. Idempotencia es de la transformación entera, no solo de la escritura.

### table-formats

**Qué es**: **Iceberg, Delta Lake y Hudi**: una capa de metadata sobre archivos (típicamente Parquet) en
object storage que agrega transacciones ACID, evolución de esquema, time travel y borrado a nivel fila —
cosas que un directorio de Parquets sueltos no te da.

**La trampa del junior**: creer que "tenemos un data lake" porque hay Parquets en un contenedor. Sin table
format no hay atomicidad: si un job muere a mitad de escritura, los lectores ven datos parciales. Y no hay
forma limpia de borrar una fila (hola, GDPR).

**Cómo lo piensa un senior**: el table format es lo que convierte un **lake** (archivos) en un **lakehouse**
(tablas). La decisión estratégica que habilita es **desacoplar el storage del motor**: la misma tabla la lee
Snowflake, Spark, Trino y Databricks sin copiar. Eso es apalancamiento real contra el vendor lock-in — y por
eso Snowflake y Databricks se pelearon tanto por Iceberg. El costo es operativo: hay que compactar, expirar
snapshots viejos y mantener la metadata, o el rendimiento se degrada solo.

**Tradeoffs reales**:

| Formato | Fuerte en | Contra |
|---|---|---|
| Apache Iceberg | Estándar abierto, soporte multi-motor amplio, partición oculta | Ecosistema más nuevo, operación de mantenimiento propia |
| Delta Lake | Integración profunda con Databricks/Spark, madurez | Históricamente centrado en Spark |
| Apache Hudi | Upserts y CDC de alta frecuencia | Menos adopción, más complejo de tunear |
| Parquet suelto (sin formato de tabla) | Simplicidad total | Sin ACID, sin deletes, sin time travel, escrituras parciales visibles |
| Tabla nativa del warehouse | Cero operación, máxima performance | Los datos viven adentro del vendor |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué agrega Iceberg sobre guardar Parquets en un bucket?*
  A: Una capa de metadata que trackea qué archivos componen la tabla en cada snapshot. Eso da commits atómicos, time travel, evolución de esquema y de partición, y borrados a nivel fila. Sin eso, la "tabla" es un directorio y cualquier escritura a medias es visible para los lectores.
- Q (senior): *¿Cuándo NO usarías un table format abierto y te quedarías con tablas nativas del warehouse?*
  A: Cuando tenés un solo motor de consulta, un equipo chico y ningún requerimiento de portabilidad. El table format abierto te cobra en operación (compactación, expiración de snapshots, tuning) y ese precio solo se paga si vas a aprovechar el multi-motor o querés poder migrar. Elegirlo "por si acaso" es complejidad sin beneficio.
- Q (trampa): *Iceberg te da ACID, ¿entonces podés usarlo como base transaccional de una app?*
  A: No. El ACID de Iceberg es a nivel de snapshot de tabla, optimizado para escrituras en lote y lecturas analíticas. No te da transacciones de baja latencia por fila ni concurrencia de miles de escritores como un OLTP. Es ACID analítico, no OLTP.

## Lo que la doc oficial cubre bien acá

- **Parquet file format** (https://parquet.apache.org/docs/file-format/) — estructura real: row groups, column chunks, pages, footer con estadísticas. Leer esto es lo que hace que "predicate pushdown" deje de ser una palabra.
- **Iceberg spec** (https://iceberg.apache.org/spec/) — metadata files, manifest lists, snapshots. Denso pero es LA fuente.
- **Azure Architecture Center — Data Guide** (https://learn.microsoft.com/azure/architecture/data-guide/) — batch vs streaming, arquitecturas de referencia, con tradeoffs explícitos.
- **dbt — How we structure our dbt projects** (https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview) — es la mejor bajada práctica y gratuita de modelado por capas que hay.

## Gaps (📕 pendiente de libros)

- **`de-lifecycle`**: el marco completo del lifecycle con sus undercurrents viene de *Fundamentals of Data Engineering* (Reis & Housley). No hay equivalente gratuito 1:1. Lo que enseñes de acá va **declarado como memoria no verificada**.
- **`dimensional-modeling`**: la referencia canónica es *The Data Warehouse Toolkit* (Kimball). Sustituto público: los *Design Tips* del Kimball Group y la guía de estructura de dbt. El proceso de diseño de 4 pasos (elegir proceso de negocio → declarar grano → identificar dimensiones → identificar hechos) se enseña con esa advertencia.
- **`columnar-storage` interno**: la spec de Parquet cubre el formato, pero el "por qué" de storage engines está en *DDIA* (Kleppmann) cap. 3. Sustituto: la spec + el paper de Snowflake del Hito 2.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `de-lifecycle`: dibujá el lifecycle completo de UN pipeline tuyo y marcá el nombre del dueño humano de cada etapa. Traeme dónde no sabés quién es el dueño — ese hueco es el hallazgo.
- `dimensional-modeling`: agarrá el mart más usado de tu empresa y escribime el grano de su tabla de hechos en UNA oración. Si necesitás dos, el grano está mezclado.
- `columnar-storage`: corré la misma query con `SELECT *` y con 3 columnas sobre una tabla grande. Traeme los bytes escaneados de ambas desde el Query Profile.
- `batch-vs-streaming`: elegí un pipeline tuyo y calculá qué costaría bajarlo de su frecuencia actual a 5 minutos. Traeme el número en créditos/mes.
- `idempotency-backfill`: corré dos veces seguidas el mismo job en un ambiente de dev. Traeme el conteo de filas antes y después. Si cambió, encontraste un bug real.
- `table-formats`: averiguá si el lake de tu empresa usa Iceberg/Delta o son Parquets sueltos, y qué pasa hoy si un job de escritura muere a mitad de camino.

### Para subir a `mastered`

- `dimensional-modeling`: diseñá un mart nuevo desde cero declarando grano, dimensiones y hechos ANTES de escribir SQL. Defendé por qué elegiste SCD Type 1 o 2 en cada dimensión. Feynman check: explicale el grano a alguien de negocio sin usar la palabra "grano".
- `idempotency-backfill`: tomá un pipeline NO idempotente de tu empresa y convertilo. Documentá qué cambiaste y probá el reproceso de una ventana. Feynman check: explicá por qué `CURRENT_DATE` adentro de la transformación rompe la idempotencia.
- `batch-vs-streaming`: escribí la recomendación formal (1 página) de si un caso real de tu empresa debería ser batch, micro-batch o streaming, con costo y latencia de cada opción. Llevala a tu equipo.
- `columnar-storage`: identificá una tabla o dataset de tu empresa con problema de archivos chicos o de escaneo excesivo, proponé la corrección y medí el antes/después.

## Anti-patterns que vas a ver en clientes reales

1. **El mart sin grano declarado**
   - Cómo se hace: alguien arma la tabla juntando lo que pidió el negocio, sin declarar qué es una fila.
   - Por qué se hace: presión de entrega, y "el dashboard ya muestra los números".
   - Costo real: las métricas se duplican al agregar por otra dimensión. El negocio pierde confianza en TODO el warehouse, no solo en esa tabla.
   - Cómo lo arregla un senior: declarar el grano, testearlo con un test de unicidad sobre la clave del grano, y separar en dos tablas si había dos granos mezclados.

2. **`CURRENT_DATE` adentro de la transformación**
   - Cómo se hace: `WHERE fecha >= CURRENT_DATE - 7` dentro de un modelo.
   - Por qué se hace: funciona perfecto la primera vez que lo corrés.
   - Costo real: el backfill produce datos distintos a los originales y nadie se entera. Los cierres históricos cambian solos.
   - Cómo lo arregla un senior: la ventana entra como parámetro desde el orquestador (`data_interval_start`) o como `var()` de dbt, con default a hoy para uso interactivo.

3. **Data lake que es un pantano de archivos**
   - Cómo se hace: cada job escribe Parquets a su carpeta, sin catálogo, sin table format, sin compactación.
   - Por qué se hace: arrancó como un experimento y nunca se formalizó.
   - Costo real: nadie sabe qué tabla es autoritativa, las lecturas son lentas por archivos chicos, y no hay forma de borrar el dato de un usuario que lo pide.
   - Cómo lo arregla un senior: catálogo + table format (Iceberg/Delta) + política de compactación y retención. Y una tabla es "la" tabla solo si está en el catálogo.

4. **ELT sin conservar el crudo**
   - Cómo se hace: se transforma en la ingesta y se descarta el payload original.
   - Por qué se hace: "ocupa lugar".
   - Costo real: cuando aparece un bug de transformación, no hay forma de reprocesar sin volver a pedir el dato a la fuente — que muchas veces ya no lo tiene.
   - Cómo lo arregla un senior: capa raw inmutable con retención definida. El storage es lo más barato de todo el stack; el dato perdido no tiene precio.

5. **Streaming por moda**
   - Cómo se hace: se monta Kafka y un procesador para alimentar un dashboard diario.
   - Por qué se hace: suena moderno y quedó lindo en la propuesta.
   - Costo real: 5x el costo, un componente más que se cae de madrugada, y un equipo que ahora necesita skills de streaming que no tenía.
   - Cómo lo arregla un senior: pregunta cuál es el costo de N minutos de atraso. Si nadie lo sabe cuantificar, es batch.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés declarar el grano de cualquier tabla de hechos en una oración, antes de escribir SQL?
- [ ] ¿Podés explicar por qué columnar gana en analítica, en términos de I/O y compresión, sin recitar la definición?
- [ ] ¿Podés decidir batch vs micro-batch vs streaming con un argumento de costo, no de moda?
- [ ] ¿Podés mirar un pipeline y detectar en 2 minutos si es idempotente o no, y decir exactamente qué línea lo rompe?
- [ ] ¿Podés explicar qué agrega un table format sobre Parquets sueltos, y cuándo NO conviene usarlo?
- [ ] En entrevista senior, ¿podés contestar "hay que reprocesar 6 meses" con un protocolo ordenado en vez de "corro el DAG con catchup"?
