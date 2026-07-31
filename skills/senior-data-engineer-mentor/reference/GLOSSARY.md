# Glosario canónico

**Para qué sirve**: fijar el **nombre exacto** de cada término. La skill usa siempre el término de acá, sin
sinónimos sueltos — si el usuario aprende dos nombres para lo mismo, aprende peor y en la entrevista duda.

**Regla de idioma**: los términos técnicos van **en inglés**, aunque la explicación esté en español. Es el
vocabulario de la documentación y de la entrevista.

**Si usás un término que no está acá y lo vas a repetir, agregalo.**

---

## Fundamentos

**Data engineering lifecycle** — Las etapas del dato: generación, ingesta, almacenamiento, transformación y serving. Atravesadas por *undercurrents*.

**Undercurrents** — Las preocupaciones que cruzan todas las etapas del lifecycle: seguridad, gestión de datos, DataOps, arquitectura, orquestación e ingeniería de software.

**ETL / ELT** — Extract-Transform-Load vs Extract-Load-Transform. La diferencia es si se transforma antes o después de cargar al destino.

**Fact table** — Tabla de hechos. Eventos medibles, con métricas numéricas y claves foráneas a dimensiones.

**Dimension table** — Tabla de dimensiones. El contexto del hecho: quién, qué, cuándo, dónde. Desnormalizada a propósito.

**Grain** (grano) — Qué representa exactamente una fila de una tabla de hechos. Primera decisión de todo diseño dimensional.

**Star schema** — Tabla de hechos rodeada de dimensiones desnormalizadas. Modelo por defecto de un mart.

**Snowflake schema** — Variante con dimensiones normalizadas en varias tablas. (Sin relación con el producto Snowflake.)

**OBT (One Big Table)** — Todo desnormalizado en una sola tabla ancha.

**SCD (Slowly Changing Dimension)** — Cómo se manejan los cambios en una dimensión. **Type 1** sobrescribe; **Type 2** versiona filas con vigencia; **Type 3** guarda el valor anterior en otra columna.

**Surrogate key** — Clave artificial de la dimensión, distinta del identificador natural. Necesaria para SCD Type 2.

**Columnar storage** — Almacenar los datos agrupados por columna en vez de por fila.

**Predicate pushdown** — Empujar el filtro hacia la capa de lectura para descartar bloques sin leerlos.

**Projection pushdown** — Leer del disco solo las columnas pedidas.

**Row group / column chunk / page** — Las unidades internas de un archivo Parquet.

**Small files problem** — La degradación de rendimiento causada por muchos archivos chicos: el overhead por archivo domina sobre la lectura de datos.

**CDC (Change Data Capture)** — Capturar los cambios de una base leyendo su log de transacciones en vez de consultarla.

**Micro-batch** — Batch de intervalo corto. El punto medio pragmático entre batch y streaming.

**Idempotencia** — Que ejecutar N veces produzca el mismo resultado que ejecutar una vez.

**Backfill** — Recomputar el pasado. Solo es seguro si el proceso es idempotente.

**Watermark** — Marca de hasta dónde se procesó, usada para decidir qué falta.

**Late-arriving data** — Datos que llegan después de que su ventana ya se procesó. La causa más común de pérdida silenciosa en incrementales.

**Upsert / merge** — Insertar si no existe, actualizar si existe. La operación que hace idempotente una escritura.

**Table format** — Capa de metadata sobre archivos que da ACID, time travel y evolución de esquema: **Iceberg**, **Delta Lake**, **Hudi**.

**Lakehouse** — Arquitectura que combina el storage barato del lake con las garantías de tabla del warehouse, vía table format.

**Data contract** — Acuerdo explícito entre productor y consumidor sobre esquema, semántica y garantías de un dato.

---

## Snowflake

**Virtual warehouse** — Cluster de compute. Se cobra por segundo mientras está encendido.

**Multi-cluster warehouse** — Warehouse que levanta instancias adicionales ante concurrencia (*scale out*).

**Scale up / scale down** — Cambiar el tamaño del warehouse. Para queries individuales pesadas.

**Scale out / scale in** — Cambiar la cantidad de clusters. Para concurrencia.

**Credit** — Unidad de facturación del compute de Snowflake.

**Micro-partition** — Bloque inmutable de datos con metadata automática (rangos por columna, cardinalidad). La unidad de pruning.

**Pruning** — Descartar micro-particiones enteras usando su metadata, sin leerlas. Reemplaza al índice.

**Clustering key** — Columnas por las que Snowflake mantiene los datos co-localizados para mejorar el pruning.

**Automatic Clustering** — El servicio en background que reordena. **Consume créditos serverless.**

**Natural clustering** — El agrupamiento que surge del orden de carga, sin declarar nada. Gratis.

**Result cache** — Cache de resultados completos en cloud services. No enciende compute.

**Metadata cache** — Estadísticas que responden ciertas consultas sin leer datos.

**Warehouse local cache** — Cache en disco del warehouse. **Se pierde al suspender.**

**Auto-suspend / auto-resume** — Apagado y encendido automático del warehouse. El auto-suspend es la palanca de costo más grande.

**Spilling** — Cuando una query no entra en memoria y derrama a disco local, o peor, a storage remoto. Señal de warehouse chico.

**Query Profile** — La herramienta de diagnóstico: particiones escaneadas vs totales, bytes, spilling, operadores.

**Zero-copy clone** — Clonar sin copiar bytes: se copian referencias a micro-particiones.

**Time Travel** — Acceso a versiones anteriores de una tabla durante un período configurable. **Consume storage.**

**Fail-safe** — Período adicional de recuperación gestionado por Snowflake, no configurable.

**ACCOUNT_USAGE** — El esquema de vistas con historial de consumo, queries y metering. De donde sale toda la atribución de costo.

**Secure Data Sharing** — Compartir datos sin copiarlos; el consumidor paga su propio compute.

---

## dbt

**Model** — Un archivo `.sql` que define una transformación. dbt lo materializa como objeto del warehouse.

**Source** — Declaración de una tabla externa que dbt no construye. Marca la frontera de entrada.

**`ref()`** — Referencia a otro modelo. Es lo que construye el DAG.

**Staging** — Capa 1:1 con la fuente: renombra, castea, limpia tipos. **Sin lógica de negocio.**

**Intermediate** — Capa de pasos reutilizables, no expuesta al consumo.

**Mart** — Capa de consumo, orientada al negocio, con grano declarado.

**Materialization** — Cómo se persiste un modelo: `view`, `table`, `incremental`, `ephemeral`, `materialized_view`, dynamic table.

**Ephemeral** — Modelo que se inyecta como CTE en quien lo referencia; no crea objeto.

**`is_incremental()`** — Macro verdadero solo cuando el modelo es incremental, ya existe y no se corre con full-refresh.

**`unique_key`** — Clave usada por la estrategia de merge para deduplicar contra el destino.

**Incremental strategy** — `merge`, `delete+insert`, `append`, `insert_overwrite`, `microbatch`.

**Lookback window** — Ventana de solapamiento que se reprocesa en cada corrida para capturar datos que llegan tarde.

**Full-refresh** — Reconstrucción completa del modelo, ignorando la lógica incremental.

**Generic test / singular test** — Test declarado en YAML y reutilizable vs test escrito como una query que no debe devolver filas.

**`dbt build`** — Ejecuta modelos, tests, seeds y snapshots en orden del DAG, deteniendo lo que depende de algo roto. Es lo que va en producción, no `dbt run`.

**Model contract** — Esquema y tipos de salida forzados y validados al construir.

**Snapshot** — Mecanismo de dbt para materializar SCD Type 2 sobre una fuente que cambia.

**Node selection** — La sintaxis de selectores: `+model`, `model+`, `tag:`, `state:modified`, `--defer`.

**`state:modified`** — Selector que compara contra un manifiesto anterior para construir solo lo que cambió. Base del CI barato.

**`--defer`** — Tomar los modelos no seleccionados desde otro ambiente (típicamente producción) en vez de reconstruirlos.

**`target/compiled/`** — Donde queda el SQL final ya resuelto. **El debugger de dbt.**

---

## Airflow

**DAG** — Directed Acyclic Graph. La definición del flujo: qué tareas hay y en qué orden.

**Task** — Una unidad de trabajo dentro del DAG.

**Operator** — Plantilla de tarea para un tipo de acción.

**Hook** — Interfaz reutilizable de conexión a un sistema externo.

**Provider** — Paquete que trae operators y hooks de una tecnología.

**TaskFlow API** — Escribir tareas como funciones Python decoradas, con paso de valores implícito.

**XCom** — Mecanismo de paso de valores entre tareas. Guardado en la metadata database. **Solo referencias, nunca datos.**

**Scheduler** — Decide qué tareas están listas y las encola.

**Executor** — Define cómo y dónde se ejecutan las tareas: Local, Celery, Kubernetes.

**Worker** — El proceso que efectivamente ejecuta la tarea.

**Triggerer** — El componente que atiende las tareas diferidas.

**Metadata database** — La base donde vive todo el estado de Airflow. Cuello de botella típico de instancias grandes.

**`data_interval_start` / `data_interval_end`** — Los límites de la ventana de datos que le toca a una corrida.

**`logical_date`** — El identificador temporal de la corrida.

**Catchup** — Si al activar el DAG se generan las corridas pendientes desde `start_date`.

**Timetable** — Regla de scheduling personalizada, más expresiva que un cron.

**Asset / Dataset** — Objeto que un DAG declara producir y otro consumir, habilitando scheduling data-aware. (La terminología varía por versión.)

**Sensor** — Tarea que espera a que se cumpla una condición.

**Deferrable operator** — Operator que libera el worker mientras espera, delegando en el triggerer.

**Pool** — Límite de tareas concurrentes sobre un recurso compartido. **Protege sistemas de afuera.**

**`max_active_runs`** — Límite de corridas simultáneas del mismo DAG.

**Top-level code** — Código en el cuerpo del archivo del DAG, fuera de las tareas. Se ejecuta en **cada parseo**.

**Dynamic task mapping** — Generar tareas en tiempo de ejecución a partir de una lista.

---

## APIs & MCP

**Resource** (en REST) — La entidad que la API expone, nombrada como sustantivo.

**Safe method** — Método que no modifica estado (GET, HEAD).

**Idempotent method** — Método cuyo efecto repetido es igual al efecto único (PUT, DELETE).

**Idempotency key** — Identificador que el cliente manda para que el servidor reconozca un reintento y no duplique la operación.

**OpenAPI** — La especificación formal del contrato de una API.

**Contract-first** — Acordar la spec antes de implementar.

**Breaking change** — Cambio que rompe consumidores existentes: sacar campos, cambiar tipos, volver obligatorio lo opcional, o cambiar el significado.

**Offset pagination** — Paginar saltando N registros. Inconsistente ante escrituras concurrentes.

**Cursor / keyset pagination** — Paginar desde un puntero a una posición del orden. Consistente.

**Total order** — Orden sin empates, necesario para que un cursor sea confiable. Típicamente fecha más identificador.

**OAuth 2.0 client credentials** — Flujo de autenticación máquina a máquina: identificador y secreto a cambio de un token de vida corta.

**Scope** — El alcance de permisos que lleva un token.

**Service principal** — Identidad de aplicación en el proveedor de nube.

**Managed identity / workload identity federation** — Identidad sin secreto persistente.

**Exponential backoff** — Esperar cada vez más entre reintentos.

**Jitter** — Variación aleatoria sobre la espera, para evitar reintentos sincronizados.

**Rate limit** — Límite de peticiones. Se comunica con 429 y una indicación de espera.

**Problem Details (RFC 9457)** — Formato estándar de cuerpo de error en HTTP.

**MCP (Model Context Protocol)** — Estándar para conectar clientes con modelos de lenguaje a servidores que exponen capacidades.

**Tool** (MCP) — Función ejecutable con efecto o costo.

**Resource** (MCP) — Dato leíble identificado por URI, incorporado como contexto.

**Prompt** (MCP) — Plantilla parametrizable que el servidor ofrece.

**Capability negotiation** — El intercambio inicial donde cliente y servidor acuerdan qué soporta cada uno.

---

## System Design & Delivery

**BFF (Backend For Frontend)** — Capa de backend dedicada a un tipo de cliente, que compone y adapta datos a su forma.

**Serving layer** — La capa por la que el dato sale del warehouse hacia sus consumidores.

**Reverse ETL** — Empujar datos del warehouse hacia sistemas operativos.

**Semantic layer** — Definiciones de métricas centralizadas, para que una métrica tenga un solo dueño.

**Stage / job / step** — La jerarquía de un pipeline de Azure DevOps.

**Service connection** — Definición protegida de acceso a un recurso externo desde un pipeline.

**Variable group** — Conjunto de variables compartidas, con enlace opcional a un almacén de secretos.

**Template** (pipeline) — Definición reutilizable. Con `extends`, se vuelve mecanismo de gobierno.

**Environment** (Azure DevOps) — Destino de despliegue con aprobaciones y verificaciones asociadas.

**Compile-time vs runtime expression** — Los distintos momentos de evaluación en un pipeline. Confundirlos es la causa número uno de variables vacías.

**Slim CI** — Construir en el pull request solo lo modificado y sus descendientes, difiriendo el resto.

**IaC (Infrastructure as Code)** — Definir la infraestructura en archivos versionados y revisables.

**Secret scanning** — Verificación automática que impide que credenciales entren al repositorio.

**RBAC (Role-Based Access Control)** — Permisos asignados a roles funcionales, no a personas.

**Lineage** — El grafo de dependencias entre datos. **Solo ve lo que pasa por la herramienta.**

**SLO de datos** — Compromiso medible sobre frescura, completitud o disponibilidad de un dato.

**Impact analysis** — Determinar quién se rompe ante un cambio. Requiere lineage **más** el log de consultas del warehouse.

**Cost attribution** — Poder responder quién consumió qué. Prerequisito de cualquier optimización de costo.
