# Catálogo de anti-patterns

**Cuándo se carga**: desde `modes/review.md`, para ampliar el detalle de un anti-pattern ya detectado con la
grilla rápida. NO se carga entero para "buscar" anti-patterns — la grilla del modo review es el índice.

**Qué es cada entrada**: slug estable (para referenciar desde un review), señal de detección concreta
(qué buscás en el código), severidad por defecto, y la corrección.

**Qué NO es**: la explicación pedagógica. Esa vive en el milestone correspondiente. Acá está la versión
operativa, para usar durante un review.

---

## Fundamentos (Hito 1)

### `grano-no-declarado`
**Severidad**: MAJOR
**Señal**: un mart sin test de unicidad sobre su clave de negocio, o cuya documentación no dice qué es una fila.
**Qué rompe**: al agregar por otra dimensión, las métricas se duplican. El negocio pierde confianza en todo el warehouse, no solo en esa tabla.
**Fix**: declarar el grano en una oración, testear unicidad sobre esa clave, y separar en dos modelos si había granos mezclados.

### `fecha-desde-adentro`
**Severidad**: CRITICAL
**Señal**: `CURRENT_DATE`, `NOW()`, `GETDATE()`, `datetime.now()` dentro de una transformación o de una tarea.
**Qué rompe**: al reprocesar el pasado, la ventana calculada es la equivocada. No hay error: solo datos incorrectos.
**Fix**: la ventana entra como parámetro externo (`{{ data_interval_start }}` en Airflow, `var()` en dbt), con default a hoy solo para uso interactivo.

### `escritura-no-idempotente`
**Severidad**: CRITICAL
**Señal**: `INSERT INTO ... SELECT` sin clave de merge, sin reemplazo de partición y sin dedupe posterior.
**Qué rompe**: el primer reintento duplica filas. Y los reintentos son inevitables.
**Fix**: `MERGE` por clave única real, o reemplazo completo de la partición correspondiente a la ventana.

### `lake-sin-formato-de-tabla`
**Severidad**: MAJOR
**Señal**: escritura directa de Parquet a un contenedor, sin table format ni catálogo.
**Qué rompe**: un job que muere a mitad deja archivos parciales visibles para los lectores; no hay forma limpia de borrar filas.
**Fix**: table format (Iceberg/Delta) más catálogo, con política de compactación y retención.

### `crudo-descartado`
**Severidad**: MAJOR
**Señal**: la ingesta transforma y no persiste el payload original.
**Qué rompe**: cuando aparece un bug de transformación, no hay forma de reprocesar sin volver a la fuente — que muchas veces ya no tiene el dato.
**Fix**: capa cruda inmutable con retención definida. El storage es lo más barato del stack.

### `streaming-por-moda`
**Severidad**: MAJOR (de diseño)
**Señal**: infraestructura de streaming alimentando un consumo que se mira una o dos veces por día.
**Qué rompe**: multiplica costo y complejidad operativa sin beneficio medible.
**Fix**: preguntar qué decisión se toma con el dato y en qué ventana. Si nadie lo puede cuantificar, es batch.

---

## Snowflake (Hito 2)

### `warehouse-siempre-encendido`
**Severidad**: MAJOR
**Señal**: `AUTO_SUSPEND` alto o desactivado en warehouses de uso intermitente.
**Qué rompe**: créditos consumidos las 24 horas por un warehouse que trabaja unas pocas. Es la fuga más común y más grande.
**Fix**: auto-suspend agresivo por defecto; si una carga necesita el cache caliente, warehouse dedicado con política propia o calentamiento programado.

### `warehouse-unico-compartido`
**Severidad**: MAJOR
**Señal**: un solo warehouse usado por ETL, BI y consultas ad-hoc.
**Qué rompe**: imposible atribuir costo, imposible dimensionar, y las cargas compiten entre sí.
**Fix**: un warehouse por carga, dimensionado y con auto-suspend propio. La atribución habilita todo lo demás.

### `pruning-anulado`
**Severidad**: MAJOR
**Señal**: filtro con función o cast sobre la columna filtrada: `WHERE YEAR(fecha) = 2025`, `WHERE UPPER(col) = 'X'`, `WHERE CAST(id AS VARCHAR) = ...`.
**Qué rompe**: se escanea la tabla entera aunque el filtro sea selectivo.
**Fix**: reescribir como rango o comparación directa sobre la columna cruda; normalizar en la carga si hace falta.

### `clustering-como-bala-de-plata`
**Severidad**: MAJOR
**Señal**: clustering keys en tablas chicas, en tablas que se recrean enteras, o sobre columnas de altísima cardinalidad.
**Qué rompe**: costo de reclustering permanente sin mejora de pruning.
**Fix**: primero el predicado, después el orden de carga, después materialización. Clustering al final, con medición de beneficio Y de costo.

### `optimizacion-medida-con-reloj`
**Severidad**: MINOR (pero invalida el trabajo)
**Señal**: se justifica una optimización con el tiempo de la segunda ejecución.
**Qué rompe**: el result cache hace que cualquier cosa parezca rápida. Se cierran tickets que no optimizaron nada.
**Fix**: medir bytes escaneados y particiones podadas, que el cache no altera.

### `time-travel-maximo-global`
**Severidad**: MAJOR
**Señal**: retención máxima configurada como default de cuenta, incluso en tablas derivadas que se recrean a diario.
**Qué rompe**: el histórico retenido multiplica el storage de esas tablas.
**Fix**: retención por tabla según su valor real de recuperación. Las tablas reconstruibles necesitan poco.

### `select-star-materializado`
**Severidad**: MAJOR
**Señal**: `SELECT *` en un modelo materializado o en una consulta programada.
**Qué rompe**: se leen y almacenan columnas que nadie usa, todos los días.
**Fix**: columnas explícitas en cualquier cosa que se materialice o corra programada.

---

## dbt (Hito 3)

### `referencia-hardcodeada`
**Severidad**: MAJOR
**Señal**: `FROM schema.tabla` en un modelo, en vez de `ref()` o `source()`.
**Qué rompe**: la dependencia desaparece del DAG — orden impredecible, lineage falso, y en dev puede apuntar a producción.
**Fix**: `ref()`/`source()` sin excepciones, con una regla de CI que falle si aparece una referencia cruda.

### `incremental-sin-lookback`
**Severidad**: MAJOR
**Señal**: `WHERE fecha > (SELECT MAX(fecha) FROM {{ this }})` sin ventana de solapamiento.
**Qué rompe**: pérdida silenciosa de todo registro que llegue con fecha anterior al máximo ya procesado.
**Fix**: ventana de lookback dimensionada con la latencia real de la fuente, o estrategia por partición, más un test de reconciliación periódico.

### `incremental-sin-clave-real`
**Severidad**: MAJOR
**Señal**: `unique_key` declarada sobre una columna que no es única en el origen, o lote entrante sin deduplicar.
**Qué rompe**: el merge falla o actualiza filas arbitrarias.
**Fix**: deduplicar en el modelo antes del merge, con criterio explícito de cuál gana.

### `todo-table-full-refresh`
**Severidad**: MINOR (MAJOR si el costo es alto)
**Señal**: materialización `table` en modelos grandes que crecen por append.
**Qué rompe**: horas de compute recreando tablas donde cambia una fracción mínima.
**Fix**: convertir a incremental los que se pagan, con las cuatro decisiones explícitas. No convertir por reflejo.

### `run-en-vez-de-build`
**Severidad**: MAJOR
**Señal**: el job productivo ejecuta `dbt run` y los tests corren aparte, o no corren.
**Qué rompe**: los datos rotos se propagan hasta el consumidor porque nada frena la cadena.
**Fix**: `dbt build` en producción, con severidades calibradas.

### `capa-violada`
**Severidad**: MINOR
**Señal**: un mart que lee `source()` directo, o un modelo de staging con lógica de negocio.
**Qué rompe**: se pierde la frontera con la fuente; un cambio de esquema impacta en lugares impredecibles.
**Fix**: staging 1:1 con la fuente y sin reglas; la lógica compartida va a la capa intermedia.

### `mart-sin-tests`
**Severidad**: MAJOR
**Señal**: un modelo de consumo sin ningún test, o solo con `not_null` decorativos.
**Qué rompe**: el primer incidente de calidad lo descubre el negocio, no el equipo.
**Fix**: unicidad sobre la clave del grano como mínimo indispensable, después freshness de la fuente e integridad referencial.

### `macros-por-todos-lados`
**Severidad**: MINOR
**Señal**: modelos que no se pueden leer sin desenrollar varios niveles de macros.
**Qué rompe**: el onboarding de cualquier persona nueva se vuelve un mes.
**Fix**: extraer al tercer uso, nunca antes, y nombrar el macro por la intención de negocio.

---

## Airflow (Hito 4)

### `encadenado-por-horario`
**Severidad**: MAJOR
**Señal**: dos DAGs con horarios escalonados "con margen" y sin dependencia declarada.
**Qué rompe**: si el primero se atrasa, el segundo corre con datos incompletos. **Sin error.**
**Fix**: dependencia por asset, trigger explícito, o unificar en un DAG.

### `catchup-con-fecha-vieja`
**Severidad**: MAJOR
**Señal**: `catchup=True` (o default) con un `start_date` muy anterior y sin `max_active_runs`.
**Qué rompe**: cientos de corridas simultáneas contra la fuente productiva al activar el DAG.
**Fix**: `catchup=False` por defecto en la plantilla del equipo, backfills explícitos y acotados, `max_active_runs` como red.

### `computo-en-el-worker`
**Severidad**: MAJOR
**Señal**: DataFrames grandes, transformaciones pesadas o joins hechos en Python dentro de la tarea.
**Qué rompe**: el worker se queda sin memoria con volumen real y el orquestador se convierte en motor mal dimensionado.
**Fix**: empujar el cómputo al warehouse; si hace falta Python sobre volumen, va en un contenedor dedicado.

### `sensor-bloqueante`
**Severidad**: MAJOR
**Señal**: sensores no diferidos, en modo que ocupa el worker, y sin timeout.
**Qué rompe**: la instancia se llena de tareas dormidas ocupando slots; parece lenta y agregar workers no ayuda.
**Fix**: deferrable, o modo que libere el worker, más timeout siempre.

### `xcom-con-datos`
**Severidad**: MAJOR
**Señal**: `return df` o cualquier payload grande devuelto por una tarea.
**Qué rompe**: la metadata database crece y se degrada para toda la instancia.
**Fix**: materializar en warehouse u object storage y pasar solo la referencia.

### `codigo-caro-en-top-level`
**Severidad**: MAJOR
**Señal**: llamadas a API, queries o lecturas pesadas en el cuerpo del módulo del DAG, fuera de cualquier tarea.
**Qué rompe**: se ejecuta en cada parseo; castiga la fuente, ralentiza el scheduler y si falla desaparece el DAG.
**Fix**: mover al momento de ejecución (mapeo dinámico) o leer desde una fuente barata de parsear.

---

## APIs & MCP (Hito 5)

### `llamada-sin-timeout`
**Severidad**: CRITICAL
**Señal**: cualquier llamada de red sin timeout explícito.
**Qué rompe**: la tarea queda colgada indefinidamente ocupando recursos, sin fallar ni alertar.
**Fix**: timeout en toda llamada, más un timeout a nivel de tarea como red de seguridad.

### `paginacion-por-offset-sobre-datos-vivos`
**Severidad**: MAJOR
**Señal**: ingesta que pagina por `offset`/`page` contra una fuente que recibe escrituras.
**Qué rompe**: duplicados y salteos silenciosos. El salteo no deja rastro.
**Fix**: cursor si la API lo permite; si no, merge por clave + ventana de solapamiento + reconciliación de conteos.

### `reintento-indiscriminado`
**Severidad**: MAJOR
**Señal**: un envoltorio de reintentos que no distingue el tipo de error.
**Qué rompe**: demora la detección del error real, quema cuota y puede disparar bloqueos por abuso.
**Fix**: transitorios (timeout, 429, 5xx) con retroceso y jitter; permanentes que fallan rápido y alertan.

### `sin-manejo-de-rate-limit`
**Severidad**: MAJOR
**Señal**: no hay tratamiento específico del 429 ni de la indicación de espera del servidor.
**Qué rompe**: la ingesta golpea hasta que la bloquean.
**Fix**: respetar lo que indica el servidor, con retroceso y jitter.

### `error-en-200`
**Severidad**: MAJOR
**Señal**: respuestas 200 con un campo de error en el cuerpo, o verbos que no corresponden a la operación.
**Qué rompe**: el error es invisible para proxies, reintentadores y monitoreo.
**Fix**: código de estado correcto más cuerpo estructurado (RFC 9457).

### `secreto-en-el-repo`
**Severidad**: CRITICAL
**Señal**: credenciales en el código, en un `.env` versionado, en el YAML del pipeline o en la definición de una conexión.
**Qué rompe**: queda en el historial para siempre; borrarla no la desactiva.
**Fix**: rotar el secreto (obligatorio), moverlo a un almacén con resolución en ejecución, y agregar escaneo de secretos al CI.

### `mcp-tool-sin-limites`
**Severidad**: CRITICAL
**Señal**: una tool MCP que ejecuta SQL arbitrario, comandos, o accede a rutas sin lista blanca.
**Qué rompe**: superficie de ataque enorme, gasto sin control, y ninguna trazabilidad.
**Fix**: tools parametrizadas con intención acotada, identidad de solo lectura sobre vistas específicas, cuotas de filas y tiempo, warehouse dedicado con presupuesto, y auditoría de cada invocación.

---

## System Design & Delivery (Hito 6)

### `front-al-warehouse`
**Severidad**: CRITICAL
**Señal**: la aplicación de cara al usuario se conecta directamente al warehouse.
**Qué rompe**: credenciales expuestas, autorización por fila no garantizable, latencia analítica en pantalla interactiva, costo que escala con los usuarios.
**Fix**: API delgada con métricas agregadas, caché y autorización en el backend; si la latencia no alcanza, tabla de servicio precalculada.

### `ci-que-construye-todo`
**Severidad**: MAJOR
**Señal**: el job de pull request ejecuta el proyecto completo.
**Qué rompe**: PRs lentos y caros. Termina en que el equipo desactiva el CI, que es el peor final.
**Fix**: construir solo lo modificado y sus descendientes, con deferral a producción, en un esquema efímero por rama.

### `ci-que-nunca-falla`
**Severidad**: MINOR (pero es una alarma)
**Señal**: el CI pasa verde siempre, desde hace meses.
**Qué rompe**: da confianza falsa. Suele significar datos no representativos, tests vacíos, o exclusión de los modelos pesados.
**Fix**: revisar qué está cubriendo realmente y con qué datos.

### `aprobacion-de-tramite`
**Severidad**: MINOR
**Señal**: un environment con aprobación donde siempre aprueba la misma persona sin información sobre el cambio.
**Qué rompe**: el control existe en el papel y no en los hechos.
**Fix**: que la aprobación muestre qué modelos cambian y qué consumidores afecta.

### `sin-catalogo-de-consumidores`
**Severidad**: MAJOR
**Señal**: nadie puede responder quién consume una tabla determinada.
**Qué rompe**: cualquier cambio es una ruleta rusa, así que en la práctica nada se cambia y la deuda se acumula.
**Fix**: lineage para lo interno, log de consultas del warehouse para lo externo, contratos explícitos para los críticos.

### `secreto-nunca-rotado`
**Severidad**: MAJOR
**Señal**: una credencial de larga vida cuya última rotación nadie puede fechar.
**Qué rompe**: es exactamente lo que un atacante busca.
**Fix**: identidad por carga de trabajo, rotación automatizada, e identidad federada donde se pueda para eliminar el secreto.

---

## Cómo agregar un anti-pattern nuevo

Cuando un review detecta algo recurrente que no está acá:

1. Slug corto y estable en kebab-case, descriptivo del problema (no de la solución).
2. Las cuatro líneas: **Severidad**, **Señal** (qué se busca en el código), **Qué rompe** (la consecuencia concreta), **Fix** (la corrección accionable).
3. Guardalo también en engram con `topic_key: skill/data-engineer-mentor/anti-pattern-discovered/{slug}` y avisale al usuario que lo agregaste.
4. Si el anti-pattern también merece explicación pedagógica, agregala al milestone correspondiente — pero acá se queda la versión operativa, corta.
