# Catálogo canónico de conceptos — 36 conceptos en 6 hitos

Este es el **catálogo finito** que la skill trackea. Cada slug es el `topic_key` usado en engram:
`skill/data-engineer-mentor/mastery/{slug}`.

Columna **Fuente primaria** = doc oficial verificado. Es lo que se linkea al cerrar un concepto.
Columna **Gimnasio (tu laburo)** = qué tarea real de tu trabajo sirve como ejercicio para ese concepto.

> **Pendiente de fuentes**: los libros canónicos (Kimball *DW Toolkit*, Kleppmann *DDIA*, Reis & Housley
> *Fundamentals of Data Engineering*) NO están cargados. Los conceptos marcados `📕 pendiente` se enseñan
> hoy con docs oficiales + sustitutos públicos, y la explicación debe **declarar explícitamente** qué parte
> viene de memoria paramétrica no verificada. Ver `playbooks/external-references.md` § "Gap de libros".

---

## Hito 1 — Fundamentos de Data Engineering (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `de-lifecycle` | 1 | Generación → ingesta → transformación → serving, con las corrientes de fondo (seguridad, orquestación, DataOps, costo) que cruzan todas las etapas | 📕 pendiente — sustituto: https://learn.microsoft.com/azure/architecture/data-guide/ | Dibujá el lifecycle de UN pipeline tuyo y marcá quién es dueño de cada etapa |
| `dimensional-modeling` | 1 | Kimball: hechos, dimensiones, grano, star schema, SCD Type 1/2 — por qué sigue siendo la base de todo mart | 📕 pendiente — sustituto: https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview | Tomá un mart existente y escribí el grano de su tabla de hechos en una oración |
| `columnar-storage` | 1 | Por qué columnar gana en analítica: compresión por columna, encoding, predicate pushdown, projection pushdown | https://parquet.apache.org/docs/file-format/ | Explicá por qué `SELECT *` te cuesta más que `SELECT 3 columnas` en Snowflake |
| `batch-vs-streaming` | 1 | Micro-batch vs streaming real vs CDC: latencia, costo, complejidad operativa, cuándo cada uno | https://learn.microsoft.com/azure/architecture/data-guide/big-data/batch-processing | Identificá un pipeline tuyo que corre cada hora y calculá qué costaría a 5 min |
| `idempotency-backfill` | 1 | Reprocesar sin duplicar ni corromper: merge/upsert, particiones por fecha, watermarks, replay | https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#re-run-dag | Corré dos veces el mismo job de tu pipeline y contá filas antes/después |
| `table-formats` | 1 | Iceberg / Delta / Hudi: capa de metadata que da ACID, time travel y schema evolution sobre object storage | https://iceberg.apache.org/spec/ | Chequeá si tu lake usa un table format o son Parquets sueltos, y qué implica |

## Hito 2 — Snowflake (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `snowflake-architecture` | 2 | Las 3 capas: storage centralizado / compute elástico multi-cluster / cloud services — y por qué esa separación cambia todo | https://docs.snowflake.com/en/user-guide/intro-key-concepts | Explicá por qué dos equipos pueden leer la misma tabla sin pelearse por recursos |
| `micro-partitions` | 2 | Bloques inmutables de 50-500MB con metadata automática; el pruning que reemplaza a los índices | https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions | Abrí un Query Profile y leé `partitions scanned` vs `partitions total` |
| `virtual-warehouses` | 2 | Sizing (XS→6XL), auto-suspend/resume, multi-cluster para concurrencia; el crédito se cobra por segundo de warehouse encendido | https://docs.snowflake.com/en/user-guide/warehouses-overview | Mirá el auto-suspend de tus warehouses productivos. ¿Está en 600s? Ahí se te va la plata |
| `snowflake-caching` | 2 | Tres cachés distintas: result cache (24h, gratis), metadata cache, warehouse local cache (SSD) | https://docs.snowflake.com/en/user-guide/querying-persisted-results | Corré la misma query dos veces y comparalas en el Query Profile |
| `clustering-pruning` | 2 | Clustering keys: cuándo mejoran el pruning, cuánto cuesta el reclustering automático, cuándo NO ponerlas | https://docs.snowflake.com/en/user-guide/tables-clustering-keys | Buscá una tabla con clustering key en tu cuenta y justificá si se paga sola |
| `snowflake-cost` | 2 | Créditos por warehouse + storage + serverless; atribución por equipo, spilling a disco, queries fugadas | https://docs.snowflake.com/en/user-guide/cost-understanding-overall | Sacá el top-10 de queries por crédito del último mes desde `ACCOUNT_USAGE` |

## Hito 3 — dbt (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `dbt-project-structure` | 3 | Las 3 capas staging → intermediate → marts, con reglas de naming y una responsabilidad por capa | https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview | Auditá tu proyecto: ¿cuántos modelos violan la capa a la que pertenecen? |
| `dbt-ref-lineage` | 3 | `ref()` y `source()` construyen el DAG; por eso dbt sabe el orden, el lineage y los ambientes | https://docs.getdbt.com/reference/dbt-jinja-functions/ref | Buscá tablas hardcodeadas (`schema.tabla`) en tu repo — cada una es un nodo perdido del DAG |
| `materializations` | 3 | view / table / incremental / ephemeral / materialized view / dynamic table: qué SQL genera cada una y qué cuesta | https://docs.getdbt.com/docs/build/materializations | Elegí un modelo `table` grande y calculá qué ahorrarías haciéndolo incremental |
| `incremental-models` | 3 | `is_incremental()`, `unique_key`, estrategias `merge`/`insert_overwrite`/`append`, datos que llegan tarde | https://docs.getdbt.com/docs/build/incremental-models | Encontrá un modelo incremental tuyo y respondé: ¿qué pasa si un registro llega 3 días tarde? |
| `dbt-tests-contracts` | 3 | Tests genéricos vs singulares, `dbt build`, contracts con constraints, packages de expectations | https://docs.getdbt.com/docs/build/data-tests | Contá cuántos modelos tuyos tienen 0 tests. Ese número es tu deuda |
| `dbt-jinja-macros` | 3 | Jinja compila ANTES de que el SQL exista: macros, `var()`, `run_query()`, y por qué `dbt compile` es tu mejor debugger | https://docs.getdbt.com/reference/dbt-jinja-functions | Agarrá un macro de tu repo y mirá su SQL compilado en `target/compiled/` |

## Hito 4 — Airflow (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `airflow-architecture` | 4 | Scheduler / executor / workers / metadata DB / triggerer / webserver: quién decide qué y dónde se rompe | https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html | Averiguá qué executor usa tu instancia y cuántos workers tiene |
| `dag-scheduling` | 4 | `data_interval`, logical date, `catchup`, cron vs timetables — la fuente #1 de confusión en Airflow | https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html | Tomá un DAG diario tuyo y decí qué fecha procesa la corrida de hoy a las 03:00 |
| `operators-hooks-taskflow` | 4 | Operators vs hooks vs providers, TaskFlow API (`@task`), XCom y sus límites de tamaño | https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html | Buscá un XCom en tu código que pase algo más grande que un id. Eso es un smell |
| `dag-idempotency` | 4 | `{{ data_interval_start }}` en vez de `datetime.now()`, tasks reintentables, backfills que no duplican | https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html | Grepeá `datetime.now()` / `current_date` en tus DAGs. Cada hit es un backfill roto |
| `airflow-assets` | 4 | Assets/datasets: scheduling data-aware en vez de cron, para que el DAG corra cuando el dato está listo | https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/assets.html | Encontrá dos DAGs tuyos encadenados por "le pongo 30 min de diferencia y listo" |
| `airflow-scaling` | 4 | Pools, `max_active_runs`, concurrency, deferrable operators, y por qué los sensors clásicos te comen los slots | https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html | Contá cuántos sensors no-deferrable tenés corriendo en paralelo |

## Hito 5 — APIs & MCP (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `rest-resource-design` | 5 | Recursos como sustantivos, métodos estándar, jerarquía, status codes con semántica real | https://cloud.google.com/apis/design | Tomá un endpoint que consumís y decí si su verbo/status son correctos |
| `api-contracts` | 5 | OpenAPI como contrato: contract-first, versionado, qué es breaking change y qué no | https://spec.openapis.org/oas/latest.html | Buscá el OpenAPI de una API que consumís. Si no existe, ese es el problema |
| `api-pagination-filtering` | 5 | Cursor vs offset, límites, orden estable — el detalle que rompe toda ingesta de API a las 100k filas | https://cloud.google.com/apis/design/design_patterns#list_pagination | Revisá una ingesta tuya de API: ¿pagina por offset? ¿qué pasa si insertan mientras leés? |
| `api-auth` | 5 | OAuth2 client credentials, OIDC, service principals, API keys, scopes y rotación | https://learn.microsoft.com/entra/identity-platform/v2-oauth2-client-creds-grant | Identificá cómo se autentica tu pipeline contra su fuente y dónde vive ese secreto |
| `api-reliability` | 5 | Idempotency keys, retries con backoff + jitter, rate limits, timeouts, errores en formato RFC 9457 | https://www.rfc-editor.org/rfc/rfc9457.html | Mirá qué hace tu ingesta ante un 429. Si no hace nada, ahí está el bug intermitente |
| `mcp-protocol` | 5 | Model Context Protocol: tools / resources / prompts, transports, capability negotiation, modelo de amenaza | https://modelcontextprotocol.io/specification | Escribí un MCP server mínimo que exponga UNA consulta a tu warehouse |

## Hito 6 — System Design & Delivery (6)

| Concepto (slug) | Hito | Descripción 1-línea | Fuente primaria | Gimnasio (tu laburo) |
|---|---|---|---|---|
| `backend-frontend-split` | 6 | Dónde vive cada responsabilidad, por qué el front no habla con el warehouse, y el patrón BFF | https://learn.microsoft.com/azure/architecture/patterns/backends-for-frontends | Dibujá quién llama a quién en un producto tuyo, y marcá dónde vive la lógica de negocio |
| `data-serving-layer` | 6 | Cómo sale el dato del warehouse: API, serving DB, reverse ETL, semantic layer — con su latencia y su costo | https://learn.microsoft.com/azure/architecture/patterns/materialized-view | Identificá cómo consume el dato la app de tu empresa y qué latencia tolera |
| `azure-pipelines` | 6 | YAML: stages/jobs/steps, templates, service connections, environments con approvals | https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/ | Leé un pipeline YAML de tu repo y marcá cuál es su gate de producción |
| `cicd-for-data` | 6 | CI de dbt con `state:modified` (slim CI), ambientes dev/staging/prod, promoción de esquemas | https://docs.getdbt.com/reference/node-selection/syntax | Averiguá si tu CI corre TODO el proyecto dbt en cada PR. Si sí, ahí se va tu plata y tu tiempo |
| `iac-secrets` | 6 | Key Vault, variable groups, service principals, IaC — y por qué el secreto nunca vive en el repo ni en el DAG | https://learn.microsoft.com/azure/devops/pipelines/library/variable-groups | Rastreá dónde está guardada la credencial de Snowflake que usa tu orquestador |
| `data-governance-cost` | 6 | RBAC, lineage, data contracts, SLA/SLO de datos, FinOps y atribución de costo por consumidor | https://docs.snowflake.com/en/user-guide/security-access-control-overview | Preguntate quién se entera primero si un mart queda desactualizado: ¿vos o el negocio? |

---

## Total: 36 conceptos

Distribución: 6 por hito × 6 hitos = **36**.

Cualquier concepto fuera de esta lista no es trackeado por la skill (por diseño — granularidad fija evita
scope creep). Si querés agregar uno, editá este archivo Y el mapa de hitos de `SKILL.md`.

## Conceptos que se confunden entre sí (usar para interleaving)

Estos tríos son donde nacen las misconceptions clásicas. Repasalos ALTERNADOS, nunca de a uno:

- `micro-partitions` + `clustering-pruning` + `snowflake-caching` — "por qué mi query es cara" tiene tres causas distintas
- `materializations` + `incremental-models` + `snowflake-cost` — la decisión de materialización ES una decisión de costo
- `dag-scheduling` + `dag-idempotency` + `idempotency-backfill` — la fecha lógica es el corazón de los tres
- `api-contracts` + `backend-frontend-split` + `mcp-protocol` — los tres son "cómo dos sistemas acuerdan una interfaz"
- `dbt-ref-lineage` + `airflow-assets` + `data-governance-cost` — tres formas de responder "¿de dónde viene este dato?"
