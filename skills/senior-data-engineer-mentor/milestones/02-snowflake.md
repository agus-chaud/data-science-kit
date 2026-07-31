# Hito 2 — Snowflake

## Por qué importa (perspectiva corporativa)

Este hito es EL hito de tu caja negra, loco. Snowflake es la herramienta que más gente usa sin entender, y
es la que más plata quema cuando no la entendés. Porque acá está el tema: Snowflake está diseñado para que
funcione sin que sepas nada. No hay índices que crear, no hay vacuum que correr, no hay particiones que
declarar. Escribís SQL y anda. **Y esa es exactamente la trampa**: anda igual de bien cuando lo usás bien
que cuando lo usás mal — lo único que cambia es la factura.

La diferencia entre un data engineer que entiende Snowflake y uno que no, se mide en dólares por mes. Bajar
la factura de Snowflake un 40% sin tocar una sola query de negocio es un logro que se cuenta en una
entrevista y que te posiciona en cualquier empresa. Y no es magia: es entender micro-particiones, warehouses
y cachés. Tres conceptos. Es así de fácil.

El otro motivo por el que este hito importa: **Snowflake es la referencia arquitectónica de la industria**.
La separación storage/compute que introdujeron en 2016 la copió todo el mundo. Si entendés por qué esa
separación cambia todo, entendés BigQuery, Databricks SQL y Redshift Serverless de arriba. Un concepto,
cuatro productos.

## Conceptos de este hito

### snowflake-architecture

**Qué es**: Tres capas desacopladas. (1) **Storage centralizado**: los datos viven una sola vez en object
storage del cloud, en formato columnar propietario. (2) **Compute (virtual warehouses)**: clusters
independientes que leen ese storage; podés tener N warehouses sobre los mismos datos. (3) **Cloud services**:
el cerebro — optimizador, metadata, autenticación, control de transacciones, result cache.

**La trampa del junior**: pensar en Snowflake como "una base de datos más grande". Con ese modelo mental no
entendés por qué apagar un warehouse no borra datos, por qué dos equipos no se pisan, por qué clonar una
tabla de 2 TB es instantáneo, ni por qué el `COUNT(*)` a veces devuelve sin encender el warehouse.

**Cómo lo piensa un senior**: la pregunta que ordena todo es **"¿esta operación toca storage, compute o
cloud services?"**. Cloud services responde metadata (COUNT, MIN/MAX, DESCRIBE, result cache) sin encender
compute — sale gratis o casi. Compute es lo que se cobra por segundo. Storage es lo más barato del stack.
Esa separación es también lo que habilita el **zero-copy clone**: clonar no copia bytes, copia referencias
a micro-particiones. Por eso un ambiente de dev completo se crea en segundos y sin costo de storage inicial.

**Tradeoffs reales**:

| Decisión | Opción A | Opción B | Criterio |
|---|---|---|---|
| Aislar cargas | Un warehouse compartido | Un warehouse por equipo/carga | Aislar da control de costo y evita contención; multiplica el overhead de warehouses tibios |
| Ambiente de dev | Base separada con copia | Zero-copy clone | Clone es instantáneo y sin costo de storage hasta que escribís. Casi siempre gana |
| Escalar | Scale UP (warehouse más grande) | Scale OUT (multi-cluster) | UP para queries individuales pesadas; OUT para muchos usuarios concurrentes. Confundirlos es el error clásico |
| Compartir datos | Copiar a la cuenta del consumidor | Secure Data Sharing | Sharing no duplica storage ni requiere pipeline; el consumidor paga su propio compute |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué significa "separación de storage y compute" y qué habilita?*
  A: Que los datos viven en un storage central independiente de los clusters que los procesan. Habilita escalar compute sin mover datos, aislar cargas de trabajo entre equipos sin duplicar información, apagar el compute sin perder nada, y clonar sin copiar.
- Q (senior): *Un `SELECT COUNT(*) FROM tabla_enorme` vuelve en 200 ms con el warehouse suspendido. ¿Cómo?*
  A: Lo resolvió la capa de cloud services desde la metadata. Snowflake mantiene el conteo de filas por micro-partición, así que un COUNT sin filtros se responde sumando metadata, sin leer datos ni encender compute. Si le agregás un `WHERE` sobre una columna no trivial, ahí sí necesita compute.
- Q (trampa): *¿Un warehouse más grande hace que todas las queries anden más rápido?*
  A: No. Duplicar el tamaño duplica el costo por segundo y duplica los recursos, pero solo ayuda si la query puede paralelizarse o si estaba haciendo spilling a disco. Una query que escanea poco y no derrama no mejora nada — pagás el doble por el mismo tiempo. Y para problemas de concurrencia (muchas queries en cola), el tamaño no ayuda: eso se resuelve con multi-cluster.

### micro-partitions

**Qué es**: Snowflake divide cada tabla automáticamente en **micro-particiones**: bloques inmutables de
aproximadamente 50-500 MB de datos sin comprimir, almacenados en columnar y comprimidos. Para cada micro-
partición guarda metadata: rango de valores por columna, cantidad de valores distintos, nulos, etc. Con esa
metadata hace **pruning**: descarta micro-particiones enteras sin leerlas.

**La trampa del junior**: buscar dónde crear el índice. No hay. Y como no hay, asume que "Snowflake escanea
todo siempre" y deja de optimizar. Peor: filtra con funciones sobre la columna (`WHERE
YEAR(fecha) = 2025`), lo cual **mata el pruning** porque la metadata está sobre `fecha`, no sobre
`YEAR(fecha)`.

**Cómo lo piensa un senior**: el pruning ES la optimización de Snowflake. Todo lo demás es secundario. La
métrica que mira un senior en el Query Profile es **`partitions scanned` / `partitions total`**: si escaneó
el 100%, no hubo pruning y ahí está el problema. Las reglas prácticas que se derivan: filtrá por la columna
cruda sin envolverla en funciones; los datos que se cargan juntos quedan juntos (el orden de carga define el
clustering natural, y por eso cargar ordenado por fecha te da pruning gratis); las micro-particiones son
**inmutables**, así que un `UPDATE` reescribe la micro-partición entera — por eso los updates masivos y
dispersos son caros.

**Tradeoffs reales**:

| Situación | Efecto en pruning | Qué hacer |
|---|---|---|
| `WHERE fecha = '2025-01-01'` | Pruning excelente si la tabla se cargó por fecha | Es el caso ideal, no toques nada |
| `WHERE YEAR(fecha) = 2025` | ❌ Pruning anulado | Reescribir como rango: `fecha >= '2025-01-01' AND fecha < '2026-01-01'` |
| `WHERE UPPER(cliente) = 'ACME'` | ❌ Pruning anulado | Normalizar en la carga, o columna derivada materializada |
| Filtro por columna de alta cardinalidad no ordenada | Pruning pobre | Evaluar clustering key (ver `clustering-pruning`) |
| Muchos `UPDATE` dispersos | Reescritura masiva de micro-particiones | Repensar como append + merge por lote |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué Snowflake no tiene índices?*
  A: Porque reemplaza el índice con metadata automática por micro-partición. En vez de una estructura auxiliar que vos mantenés, guarda min/max y cardinalidad de cada bloque y descarta bloques enteros al planificar. Menos operación para vos, pero significa que la optimización pasa por cómo están ORDENADOS los datos, no por qué índice creaste.
- Q (senior): *Una query filtra por una columna y escanea igual el 100% de las particiones. ¿Qué revisás?*
  A: Primero, si el filtro envuelve la columna en una función o un cast — eso anula el pruning. Segundo, si la columna no correlaciona con el orden físico de carga (típico en columnas de alta cardinalidad cargadas desordenadas). Tercero, si el predicado viene de una subquery o un join que el optimizador no puede empujar. La corrección va en ese orden: primero reescribir el predicado, después evaluar clustering, y clustering solo si la tabla es grande y el patrón es estable.
- Q (trampa): *¿Conviene hacer `UPDATE` de una columna en una tabla de 5 mil millones de filas?*
  A: Casi nunca. Las micro-particiones son inmutables: actualizar una fila reescribe la micro-partición completa que la contiene. Un update disperso sobre una tabla enorme puede terminar reescribiendo casi toda la tabla, con el costo de compute y de storage (las versiones viejas quedan por Time Travel). El patrón correcto suele ser recrear la tabla con `CREATE OR REPLACE ... AS SELECT` o hacer merges por lote acotados por partición.

### virtual-warehouses

**Qué es**: Clusters de compute que ejecutan las queries. Tamaños desde XS y cada escalón **duplica**
recursos y costo por hora. Se cobra **por segundo mientras están encendidos** (con un mínimo por arranque),
no por query ni por dato escaneado. Tienen `AUTO_SUSPEND` y `AUTO_RESUME`, y pueden configurarse
**multi-cluster** para escalar horizontalmente ante concurrencia.

**La trampa del junior**: dejar el `AUTO_SUSPEND` alto (o desactivado) porque "así no hay latencia de
arranque". Ese warehouse queda encendido sin trabajo, quemando créditos las 24 horas. Es, lejos, la fuga de
plata más común en cuentas de Snowflake. La otra trampa: agrandar el warehouse cuando las queries hacen cola
— eso no es problema de tamaño, es de concurrencia.

**Cómo lo piensa un senior**: el warehouse es un **grifo de plata** que vos abrís y cerrás. Dos preguntas
distintas: *"¿mis queries individuales son lentas?"* → tamaño (scale up). *"¿mis queries esperan en cola?"*
→ multi-cluster (scale out). Confundirlas cuesta caro en las dos direcciones. Y una regla que sorprende a
todos: **un warehouse del doble de tamaño que corre la mitad del tiempo cuesta lo mismo** — así que si la
query escala bien, subir de tamaño es gratis en costo y gratis en tiempo ganado. Por eso "el warehouse
grande es caro" es una media verdad: es caro si NO escala.

**Tradeoffs reales**:

| Síntoma | Causa probable | Acción | Lo que NO hay que hacer |
|---|---|---|---|
| Query lenta, spilling a disco en el profile | Warehouse chico para el volumen | Scale up (tamaño mayor) | Agregar clusters |
| Queries en cola, cada una rápida | Concurrencia | Multi-cluster (scale out) | Agrandar el warehouse |
| Costo alto sin uso | `AUTO_SUSPEND` largo o apagado | Bajar auto-suspend agresivamente | Achicar el warehouse (no es el problema) |
| Primera query del día lenta | Cache local frío | Aceptarlo, o warm-up programado | Desactivar auto-suspend |
| Un equipo afecta a otro | Warehouse compartido | Separar warehouses por carga | Subir el tamaño para "que alcance" |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cuál es la diferencia entre agrandar un warehouse y agregarle clusters?*
  A: Agrandar (scale up) le da más recursos a cada query — sirve cuando una query es lenta o hace spilling. Agregar clusters (scale out, multi-cluster) levanta instancias adicionales para atender más queries en paralelo — sirve cuando hay cola por concurrencia. Son problemas distintos y la solución cruzada no funciona.
- Q (senior): *¿Cómo bajás la factura de Snowflake sin tocar una query de negocio?*
  A: En este orden: (1) auditar `AUTO_SUSPEND` de todos los warehouses y bajarlo — es la ganancia más grande y de menor riesgo; (2) separar warehouses por carga para poder atribuir costo y dimensionar cada uno; (3) identificar el top de queries por crédito en `ACCOUNT_USAGE.QUERY_HISTORY` y ver si hay tablas sin pruning o materializaciones mal elegidas; (4) revisar tareas y pipelines que corren más frecuente de lo que el negocio consume; (5) recién ahí, tuning fino de clustering. Los primeros dos puntos suelen dar la mayor parte del ahorro.
- Q (trampa): *Un warehouse L cuesta el doble que un M. ¿Entonces conviene siempre el M?*
  A: No, porque el cobro es por tiempo encendido. Si la query escala linealmente, el L la corre en la mitad de tiempo y el costo total es idéntico — pero terminás antes y liberás el warehouse. El L sale más caro solo si la query NO escala (poca paralelización, mucho skew). La pregunta correcta es "¿esta carga escala?", medida con el tiempo real de ejecución en ambos tamaños, no con el precio por hora.

### snowflake-caching

**Qué es**: Tres cachés distintas que se confunden todo el tiempo. (1) **Result cache**: en cloud services,
guarda el resultado completo de una query; si repetís exactamente la misma query y los datos no cambiaron,
devuelve sin encender compute. (2) **Metadata cache**: estadísticas de micro-particiones que responden
COUNT/MIN/MAX sin leer datos. (3) **Warehouse local cache (SSD)**: los datos que ese warehouse ya leyó
quedan en su disco local — **y se pierden cuando el warehouse se suspende**.

**La trampa del junior**: ver que la segunda ejecución fue instantánea y concluir "ya está optimizada". No
está optimizada: está cacheada. En cuanto cambie un byte de la tabla o le cambies una coma a la query, se
vuelve a la lentitud original. Y el usuario final, que corre variaciones de la query, nunca ve ese cache.

**Cómo lo piensa un senior**: el cache es **medición contaminada**. Para saber si una query realmente
mejoró, un senior compara *bytes escaneados y particiones podadas*, no tiempo de reloj. Además entiende la
tensión operativa: bajar `AUTO_SUSPEND` ahorra créditos pero tira el cache local, así que la primera query
después de cada suspensión paga el precio del disco frío. Ese es un tradeoff real de costo vs latencia que
se decide por carga: para un warehouse de ETL nocturno, suspender agresivo. Para uno que sirve un dashboard
que se consulta cada 10 minutos, tal vez convenga tolerar unos minutos de espera.

**Tradeoffs reales**:

| Cache | Dónde vive | Cuándo se invalida | Cuesta compute |
|---|---|---|---|
| Result cache | Cloud services | Cambian los datos subyacentes, o cambia el texto de la query, o hay funciones no determinísticas | ❌ No |
| Metadata cache | Cloud services | Con cada DML sobre la tabla | ❌ No |
| Warehouse local (SSD) | Disco del warehouse | Al suspender el warehouse, o por desalojo | ✅ Sí (pero menos I/O remoto) |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué la misma query vuelve instantánea la segunda vez?*
  A: Probablemente el result cache: Snowflake guarda el resultado y si repetís exactamente la misma consulta sobre datos sin cambios, te lo devuelve sin usar compute. Si el texto de la query cambia aunque sea en un espacio, o si la tabla recibió un DML, se recalcula.
- Q (senior): *Querés medir si tu optimización sirvió. ¿Cómo evitás que el cache te mienta?*
  A: No me guío por el tiempo de reloj. Miro en el Query Profile los bytes escaneados y `partitions scanned / partitions total`, que son independientes del cache de resultado. Si necesito medir tiempo real, corro con el cache de resultado desactivado a nivel sesión y sobre un warehouse recién iniciado, para que tampoco intervenga el cache local.
- Q (trampa): *Bajar `AUTO_SUSPEND` a 60 segundos, ¿siempre ahorra plata?*
  A: Casi siempre ahorra créditos de warehouse ocioso, pero tiene un costo escondido: al suspender se pierde el cache local en SSD, así que las siguientes queries releen desde storage remoto y tardan más — y ese tiempo extra también se cobra. Para cargas espaciadas, suspender agresivo gana claramente. Para cargas frecuentes sobre las mismas tablas grandes, conviene medir: a veces un auto-suspend algo mayor sale más barato en total.

### clustering-pruning

**Qué es**: Una **clustering key** le dice a Snowflake por qué columnas mantener los datos co-localizados
físicamente, para mejorar el pruning en tablas grandes donde el orden natural de carga ya no ayuda.
Snowflake reordena en background con **Automatic Clustering**, y eso **consume créditos serverless**.

**La trampa del junior**: tratar la clustering key como si fuera un índice y ponerla en todas las tablas
"por las dudas". Resultado: pagás reclustering permanente en tablas donde no cambia nada el pruning. La
otra trampa: elegir una columna de altísima cardinalidad (como un UUID) como clustering key — eso no
agrupa nada útil y el costo de mantenerlo es alto.

**Cómo lo piensa un senior**: clustering es la **última** palanca, no la primera. El orden de intervención
es: (1) arreglar predicados que anulan el pruning, (2) revisar si la carga puede hacerse ordenada por la
columna de filtro (que da clustering natural gratis), (3) evaluar materializaciones alternativas, y (4)
recién ahí clustering key — y solo en tablas grandes (típicamente multi-TB), con un patrón de filtrado
estable en el tiempo y una relación costo/beneficio medida con `SYSTEM$CLUSTERING_INFORMATION`. Si la tabla
se reescribe entera todos los días, no tiene sentido: cada carga ordenada ya te da el orden.

**Tradeoffs reales**:

| Situación | ¿Clustering key? | Por qué |
|---|---|---|
| Tabla chica o mediana | ❌ No | El pruning natural alcanza; el costo de reclustering no se paga |
| Tabla grande, filtro estable por 1-2 columnas de cardinalidad media | ✅ Sí, candidata | Es el caso de libro |
| Tabla grande recreada full cada corrida | ❌ No | Ordená en el `INSERT`/`CREATE` y obtenés el orden gratis |
| Clustering key sobre UUID / alta cardinalidad | ❌ No | No agrupa; costo alto, beneficio nulo |
| Muchos DML dispersos sobre la tabla | ⚠️ Cuidado | El reclustering se dispara seguido y el costo se vuelve permanente |
| Filtro por fecha en tabla append-only cargada por fecha | ❌ No hace falta | Ya tenés clustering natural por orden de carga |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué hace una clustering key?*
  A: Le indica a Snowflake por qué columnas mantener los datos físicamente agrupados, para que la metadata de micro-particiones permita descartar más bloques al filtrar por esas columnas. No es un índice: no hay estructura auxiliar, se reordenan los datos.
- Q (senior): *¿Cómo justificás poner o sacar una clustering key?*
  A: Con números de los dos lados. Del lado del beneficio: `partitions scanned / total` y bytes escaneados de las queries representativas, antes y después. Del lado del costo: los créditos de Automatic Clustering que la tabla consume por mes, visibles en `ACCOUNT_USAGE`. Si el ahorro de compute en las consultas no supera el costo de reclustering, se saca. Y antes de todo eso reviso `SYSTEM$CLUSTERING_INFORMATION` para ver si la tabla realmente está mal agrupada o si el problema es el predicado.
- Q (trampa): *¿La clustering key acelera todas las queries de esa tabla?*
  A: Solo las que filtran por las columnas de la key (o por un prefijo de ellas). Una query que filtra por otra columna no se beneficia en nada y sigue pagando su escaneo, mientras la tabla entera paga el costo de reclustering. Por eso la key se elige a partir del patrón real de consulta, no del criterio "la columna más importante del negocio".

### snowflake-cost

**Qué es**: La factura tiene tres componentes: **compute de warehouses** (créditos por segundo encendido),
**storage** (por TB/mes, incluye las versiones retenidas por Time Travel y Fail-safe) y **serverless**
(Automatic Clustering, Snowpipe, tareas serverless, materialized views, búsqueda). En la práctica el compute
domina.

**La trampa del junior**: mirar el total de la factura y no poder responder QUIÉN lo gastó. Sin separación
por warehouse ni tags, el costo es una bola opaca y la única palanca que queda es "usen menos", que no es
una palanca.

**Cómo lo piensa un senior**: **la atribución es prerequisito de la optimización**. Primero separás
warehouses por equipo/carga para que el costo tenga dueño, después medís desde `ACCOUNT_USAGE`
(`QUERY_HISTORY`, `WAREHOUSE_METERING_HISTORY`) y recién ahí optimizás lo que más pesa. Un senior también
sabe que el storage rara vez es el problema — salvo cuando alguien puso Time Travel en 90 días sobre tablas
que se reescriben enteras a diario, y ahí el histórico multiplica el storage por un factor absurdo.

**Tradeoffs reales**:

| Palanca | Ahorro típico | Riesgo | Esfuerzo |
|---|---|---|---|
| Bajar `AUTO_SUSPEND` | Alto | Bajo (cache frío) | Mínimo |
| Separar warehouses por carga | Habilita todo lo demás | Bajo | Bajo |
| Bajar frecuencia de pipelines a lo que el negocio consume | Alto | Medio (acordar SLA) | Bajo |
| Convertir modelos full-refresh a incrementales | Alto en tablas grandes | Medio (complejidad, late data) | Medio |
| Ajustar Time Travel por tabla | Medio (storage) | Medio (menos margen de recuperación) | Bajo |
| Tuning de clustering | Bajo-medio | Medio (costo de reclustering) | Alto |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cómo se cobra Snowflake?*
  A: Principalmente por créditos de compute, que se consumen por segundo mientras un warehouse está encendido — no por query ni por bytes escaneados. Aparte se paga storage por TB/mes (incluyendo Time Travel y Fail-safe) y features serverless.
- Q (senior): *Te dan una cuenta de Snowflake con la factura duplicada respecto al mes pasado. ¿Primeros pasos?*
  A: Comparo `WAREHOUSE_METERING_HISTORY` mes contra mes para ubicar QUÉ warehouse creció — eso ya me dice de qué equipo o proceso es. Con el warehouse identificado, voy a `QUERY_HISTORY` filtrado por ese warehouse y ordeno por tiempo de ejecución total: busco queries nuevas, queries que crecieron, o un cambio de frecuencia en un pipeline. En paralelo miro si alguien cambió tamaños de warehouse o auto-suspend. En la mayoría de los casos es una de tres cosas: un pipeline que empezó a correr más seguido, un modelo que pasó de incremental a full, o un warehouse que quedó encendido.
- Q (trampa): *¿El storage barato significa que Time Travel es gratis?*
  A: No. Time Travel retiene las versiones anteriores de las micro-particiones, así que en una tabla que se reescribe completa todos los días, 90 días de retención pueden multiplicar el storage de esa tabla por decenas. Sumale Fail-safe, que es adicional y no configurable. La retención se define por tabla según su valor real de recuperación, no como default global.

## Lo que la doc oficial cubre bien acá

- **Key Concepts & Architecture** (https://docs.snowflake.com/en/user-guide/intro-key-concepts) — las tres capas, explicado corto y claro. Es el punto de entrada.
- **Micro-partitions & Data Clustering** (https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions) — la página que más devuelve por minuto leído en toda la doc de Snowflake.
- **Warehouses overview + considerations** (https://docs.snowflake.com/en/user-guide/warehouses-overview) — sizing, multi-cluster, auto-suspend con criterios.
- **Cost management** (https://docs.snowflake.com/en/user-guide/cost-understanding-overall) — desglose de qué se cobra y desde qué vistas medirlo.
- **The Snowflake Elastic Data Warehouse (SIGMOD 2016)** — el paper de los creadores. Acá está el POR QUÉ de la arquitectura, no el cómo. Si querés cerrar la caja negra de verdad, es esto.

## Gaps (📕 pendiente de libros)

- **Storage engines en general**: por qué columnar, cómo funcionan los formatos de bloque y las estadísticas — está en *DDIA* (Kleppmann) cap. 3. Sustituto: el paper de Snowflake + la spec de Parquet del Hito 1.
- No hay gap grande de libro en este hito: la doc oficial de Snowflake es densa y buena. El gap real es **de práctica**, y se cubre con el gimnasio de tu laburo.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `snowflake-architecture`: hacé un zero-copy clone de una base productiva a un esquema de dev y cronometralo. Traeme el tiempo y el tamaño de la base.
- `micro-partitions`: abrí el Query Profile de una query lenta tuya y traeme `partitions scanned` vs `partitions total`. Después reescribí un predicado que anule el pruning (si encontrás uno) y traeme el número nuevo.
- `virtual-warehouses`: listá todos los warehouses de tu cuenta con su tamaño y su `AUTO_SUSPEND`. Traeme la lista. Ese solo ejercicio suele encontrar plata tirada.
- `snowflake-caching`: corré la misma query tres veces — con warehouse frío, con warehouse caliente, y repetida idéntica. Traeme los tres tiempos y explicá cuál cache actuó en cada caso.
- `clustering-pruning`: buscá si hay tablas con clustering key en tu cuenta y corré `SYSTEM$CLUSTERING_INFORMATION` sobre una. Traeme el output y decime si se justifica.
- `snowflake-cost`: sacá de `ACCOUNT_USAGE.QUERY_HISTORY` el top-10 de queries por tiempo de ejecución del último mes. Traeme la lista y quién las corre.

### Para subir a `mastered`

- `virtual-warehouses` + `snowflake-cost`: armá una propuesta de reducción de costo para tu cuenta real, con el ahorro estimado por palanca y el riesgo de cada una. Presentala a tu equipo. Feynman check: explicale a alguien de finanzas por qué un warehouse más grande puede salir lo mismo.
- `micro-partitions`: tomá una query cara de producción y bajale los bytes escaneados de forma medible, sin cambiar el resultado. Documentá el antes/después. Feynman check: explicá el pruning sin usar la palabra "índice" ni la palabra "partición".
- `clustering-pruning`: evaluá con datos si una tabla grande de tu empresa debería tener clustering key, incluyendo el costo mensual de reclustering. Escribí la recomendación con el número de las dos columnas.
- `snowflake-architecture`: diseñá el esquema de warehouses de tu organización (cuáles, de qué tamaño, para qué carga, con qué auto-suspend) y defendé la separación.

## Anti-patterns que vas a ver en clientes reales

1. **El warehouse que nunca se apaga**
   - Cómo se hace: `AUTO_SUSPEND` alto o desactivado "para que no haya latencia".
   - Por qué se hace: alguien se quejó de que la primera query tardaba y esa fue la solución rápida.
   - Costo real: créditos las 24 horas por un warehouse que trabaja 2. Es la fuga más común y más grande.
   - Cómo lo arregla un senior: auto-suspend agresivo por default, y si una carga puntual necesita calor, un warm-up programado o un warehouse dedicado con política distinta.

2. **Un solo warehouse para toda la empresa**
   - Cómo se hace: se crea `COMPUTE_WH` al inicio y todo el mundo lo usa.
   - Por qué se hace: inercia. Nunca hubo un momento explícito de decidir lo contrario.
   - Costo real: imposible atribuir costo, imposible dimensionar, y el ETL nocturno compite con los dashboards del directorio.
   - Cómo lo arregla un senior: un warehouse por carga (ETL, BI, ad-hoc, data science), dimensionado y con auto-suspend propio. La atribución habilita todo lo demás.

3. **Clustering key como bala de plata**
   - Cómo se hace: la query anda lenta, alguien le pone una clustering key.
   - Por qué se hace: es la única palanca que suena a "optimización de base de datos".
   - Costo real: créditos de reclustering permanentes sin mejora de pruning, porque el problema real era un predicado con función encima de la columna.
   - Cómo lo arregla un senior: primero el predicado, después el orden de carga, después materialización, y clustering al final con medición de las dos columnas (beneficio y costo).

4. **Medir optimizaciones con el reloj**
   - Cómo se hace: "la cambié y ahora tarda 200 ms, quedó buenísima".
   - Por qué se hace: el result cache es invisible si no sabés que existe.
   - Costo real: se cierran tickets de optimización que no optimizaron nada, y el problema vuelve en producción con queries ligeramente distintas.
   - Cómo lo arregla un senior: mide bytes escaneados y particiones podadas, que el cache no altera.

5. **Time Travel de 90 días como default global**
   - Cómo se hace: se configura al máximo "por seguridad" en toda la cuenta.
   - Por qué se hace: parece gratis porque el storage es barato.
   - Costo real: en tablas que se reescriben enteras a diario, el histórico retenido multiplica el storage de forma brutal.
   - Cómo lo arregla un senior: retención por tabla según su valor real de recuperación. Tablas derivadas y reconstruibles necesitan poco; tablas raw irrecuperables necesitan más.

6. **`SELECT *` en modelos productivos**
   - Cómo se hace: se copia el patrón de una exploración ad-hoc a un modelo que corre todos los días.
   - Por qué se hace: comodidad, y en tiempo de desarrollo no se nota.
   - Costo real: se leen y materializan columnas que nadie usa, en columnar eso es directamente I/O y storage tirados, todos los días.
   - Cómo lo arregla un senior: columnas explícitas en cualquier modelo que se materialice o corra programado.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés explicar las tres capas y decir, para una operación cualquiera, cuál interviene y si cuesta compute?
- [ ] ¿Podés leer un Query Profile y decir en 30 segundos si hubo pruning y si hubo spilling?
- [ ] ¿Podés decidir entre scale up y scale out a partir del síntoma, sin dudar?
- [ ] ¿Podés distinguir las tres cachés y explicar cuál te está mintiendo cuando medís una optimización?
- [ ] ¿Podés justificar poner o sacar una clustering key con números de beneficio Y de costo?
- [ ] ¿Podés armar un plan de reducción de costo ordenado por impacto/esfuerzo sin tocar queries de negocio?
- [ ] En entrevista senior, ¿podés contestar "la factura se duplicó, ¿qué hacés?" con un protocolo de diagnóstico en vez de una lista de tips?
