# External References — Fuentes oficiales curadas

**Cuándo se carga**: lazy, cuando el mentor necesita apuntar a la fuente canónica de un concepto, o cuando
el usuario pide material para estudiar.

**Regla dura**: ningún link inventado. Si no estás seguro del URL exacto, escribí la pista de búsqueda
("buscar en docs.snowflake.com: {tema}") en vez de mandar a una página fantasma. Un link falso destruye la
confianza, y la confianza es todo.

**Segunda regla**: las herramientas de este dominio se mueven por versión. Antes de citar una página de dbt
o Airflow, verificá que corresponda a la versión que usa el usuario.

---

## Las dos capas

Este archivo tiene dos secciones que **no se mezclan**, porque se consultan en momentos distintos:

| Capa | Qué responde | Cuándo se consulta |
|---|---|---|
| **Capa concepto** | "¿Por qué está diseñado así?" | Estudiando, antes de decidir |
| **Capa sintaxis** | "¿Cómo se escribe esto?" | Con el editor abierto, mientras escribís |

Para un usuario que YA usa las herramientas, la capa concepto es la que devuelve. Ofrecé sintaxis solo
cuando la pregunta es operativa.

---

# CAPA CONCEPTO

## Snowflake

### Snowflake — Key Concepts & Architecture
**Tipo:** docs oficiales
**URL:** https://docs.snowflake.com/en/user-guide/intro-key-concepts
**Qué cubre:** las tres capas (storage, compute, cloud services) y el vocabulario base.
**Qué tenés que entender:**
- Por qué la separación storage/compute permite apagar compute sin perder datos
- Qué resuelve la capa de cloud services sin encender warehouse
- Cómo se relaciona con zero-copy clone y con data sharing
**Tiempo de lectura:** 45 min
**Cuándo volver:** cuando algo del comportamiento de Snowflake no cierre con tu modelo mental.

### Snowflake — Micro-partitions & Data Clustering
**Tipo:** docs oficiales
**URL:** https://docs.snowflake.com/en/user-guide/tables-clustering-micropartitions
**Qué cubre:** cómo se dividen las tablas, qué metadata guarda cada bloque, cómo funciona el pruning y qué hace una clustering key.
**Qué tenés que entender:**
- Qué es una micro-partición y qué metadata trae
- Cómo el pruning reemplaza al índice
- Qué predicados anulan el pruning
- Cuándo una clustering key se paga y cuándo no
**Tiempo de lectura:** 1 hora
**Cuándo volver:** es LA página de Snowflake. Volvé cada vez que optimices una query.

### The Snowflake Elastic Data Warehouse (SIGMOD 2016)
**Tipo:** paper de los creadores
**URL:** buscar "The Snowflake Elastic Data Warehouse SIGMOD 2016" — hay PDF público y también está en la ACM Digital Library
**Qué cubre:** el diseño original y el razonamiento detrás de la arquitectura multi-cluster shared-data.
**Qué tenés que entender:**
- Por qué eligieron separar storage de compute (y qué problema de los warehouses previos resolvía)
- Por qué no hay índices ni estadísticas manuales
- Cómo pensaron la elasticidad y el aislamiento de cargas
**Tiempo de lectura:** 2 horas
**Cuándo volver:** una vez, bien leído. Es el material que más cierra la caja negra de todo el hito 2.

### Snowflake — Warehouses overview
**Tipo:** docs oficiales
**URL:** https://docs.snowflake.com/en/user-guide/warehouses-overview
**Qué cubre:** sizing, auto-suspend/resume, multi-cluster, consideraciones de carga.
**Qué tenés que entender:** cuándo scale up vs scale out; el efecto real del auto-suspend sobre el costo y sobre el cache local.
**Tiempo de lectura:** 45 min
**Cuándo volver:** al dimensionar warehouses o al atacar la factura.

### Snowflake — Cost management
**Tipo:** docs oficiales
**URL:** https://docs.snowflake.com/en/user-guide/cost-understanding-overall
**Qué cubre:** qué se cobra (compute, storage, serverless) y desde qué vistas de `ACCOUNT_USAGE` medirlo.
**Qué tenés que entender:** cómo atribuir consumo por warehouse y por query; el impacto de Time Travel en storage.
**Tiempo de lectura:** 1 hora
**Cuándo volver:** cada vez que alguien pregunte por la factura.

## dbt

### dbt — How we structure our dbt projects
**Tipo:** guía oficial de best practices
**URL:** https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview
**Qué cubre:** las capas staging / intermediate / marts, naming, organización por dominio.
**Qué tenés que entender:**
- La responsabilidad única de cada capa
- Por qué staging es 1:1 con la fuente y sin lógica de negocio
- Cuándo aparece la capa intermedia
**Tiempo de lectura:** 2 horas
**Cuándo volver:** es la mejor guía gratuita de criterio de modelado que existe. Volvé al refactorizar.

### dbt — Materializations e Incremental models
**Tipo:** docs oficiales
**URL:** https://docs.getdbt.com/docs/build/materializations y https://docs.getdbt.com/docs/build/incremental-models
**Qué cubre:** cada materialización, las estrategias incrementales y sus configuraciones.
**Qué tenés que entender:**
- Qué SQL genera cada materialización
- Cuándo aplica cada estrategia incremental
- Cómo se comporta `is_incremental()` y qué pasa con `--full-refresh`
**Tiempo de lectura:** 1.5 horas
**Cuándo volver:** antes de convertir cualquier modelo a incremental.

### dbt — Data tests y Model contracts
**Tipo:** docs oficiales
**URL:** https://docs.getdbt.com/docs/build/data-tests y https://docs.getdbt.com/docs/collaborate/govern/model-contracts
**Qué cubre:** tests genéricos y singulares, severidad, contracts con constraints.
**Qué tenés que entender:** la diferencia entre `run` y `build`; qué garantiza un contract y qué no.
**Tiempo de lectura:** 1 hora
**Cuándo volver:** al definir la estrategia de calidad de un mart nuevo.

### dbt Developer Blog
**Tipo:** blog oficial
**URL:** https://docs.getdbt.com/blog
**Qué cubre:** el razonamiento detrás de las decisiones de diseño de dbt y patrones de la comunidad.
**Qué tenés que entender:** es donde está el "por qué" que la doc de referencia no explica.
**Tiempo de lectura:** variable, 20-40 min por artículo
**Cuándo volver:** cuando una convención de dbt te parezca arbitraria.

### dbt Fundamentals (curso oficial gratuito)
**Tipo:** curso
**URL:** https://courses.getdbt.com
**Qué cubre:** el recorrido completo guiado, con ejercicios.
**Cuándo volver:** si venís de SQL suelto y querés la base ordenada en un fin de semana.

## Airflow

### Airflow — Core Concepts
**Tipo:** docs oficiales
**URL:** https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html
**Qué cubre:** arquitectura, DAGs, tareas, operators, XCom.
**Qué tenés que entender:** qué hace cada componente y dónde puede romperse cada uno.
**Tiempo de lectura:** 1.5 horas
**Cuándo volver:** al diagnosticar problemas de la instancia, no de un DAG.

### Airflow — Authoring and Scheduling
**Tipo:** docs oficiales
**URL:** https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html
**Qué cubre:** intervalos de datos, catchup, timetables, assets, deferring.
**Qué tenés que entender:** el modelo de ventana de datos. **Es la sección que resuelve el malentendido central de Airflow.**
**Tiempo de lectura:** 2 horas
**Cuándo volver:** cada vez que dudes de qué procesa una corrida.

### Airflow — Templates Reference
**Tipo:** docs oficiales
**URL:** https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html
**Qué cubre:** todas las variables templadas disponibles en tareas.
**Qué tenés que entender:** cuáles corresponden al intervalo y cuáles al momento de ejecución. Confundirlas es el bug clásico de backfill.
**Tiempo de lectura:** 30 min
**Cuándo volver:** cada vez que escribas un filtro por fecha en un DAG.

### Airflow — Best Practices
**Tipo:** docs oficiales
**URL:** https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html
**Qué cubre:** código de nivel superior, idempotencia, testing de DAGs, gestión de recursos.
**Tiempo de lectura:** 1 hora
**Cuándo volver:** antes de escribir tu primer DAG productivo, y al hacer code review de DAGs ajenos.

### Astronomer Learn
**Tipo:** guías de la empresa que mantiene gran parte de Airflow
**URL:** https://docs.astronomer.io/learn
**Qué cubre:** prácticamente todo Airflow, explicado mejor que la doc oficial.
**Qué tenés que entender:** cuando la doc oficial es árida, esto es lo que se lee. **Es el mejor material pedagógico de Airflow que existe.**
**Tiempo de lectura:** variable
**Cuándo volver:** siempre que un tema de Airflow no cierre.

### Airflow Improvement Proposals (AIPs)
**Tipo:** design docs
**URL:** buscar "Airflow Improvement Proposals" en el wiki oficial de Apache Airflow (cwiki.apache.org)
**Qué cubre:** el razonamiento detrás de cada cambio grande del proyecto.
**Cuándo volver:** cuando quieras entender POR QUÉ Airflow hace algo de una forma que parece rara.

## Azure DevOps & System Design

### Azure Architecture Center — Cloud Design Patterns
**Tipo:** catálogo oficial de patrones
**URL:** https://learn.microsoft.com/azure/architecture/patterns/
**Qué cubre:** decenas de patrones de arquitectura con problema, solución, consideraciones y cuándo NO usarlos.
**Qué tenés que entender:** al menos BFF, Materialized View, Retry, Circuit Breaker, Cache-Aside, Claim-Check.
**Tiempo de lectura:** 20 min por patrón
**Cuándo volver:** al diseñar cualquier integración. Es gratis, es aplicable fuera de Azure, y da vocabulario para defender decisiones.

### Azure Well-Architected Framework
**Tipo:** framework oficial
**URL:** https://learn.microsoft.com/azure/well-architected/
**Qué cubre:** los cinco pilares (confiabilidad, seguridad, costo, excelencia operativa, eficiencia de performance).
**Qué tenés que entender:** es el vocabulario con el que se discuten decisiones ante management. Sirve tanto para defender una propuesta como para auditar una existente.
**Tiempo de lectura:** 3 horas
**Cuándo volver:** antes de presentar una decisión de arquitectura a alguien que no es técnico.

### Azure Pipelines — YAML schema
**Tipo:** docs oficiales
**URL:** https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/
**Qué cubre:** todas las claves válidas de un pipeline.
**Cuándo volver:** al escribir o depurar un pipeline.

### Azure Pipelines — Expressions
**Tipo:** docs oficiales
**URL:** https://learn.microsoft.com/azure/devops/pipelines/process/expressions
**Qué cubre:** los distintos tipos de expresión y en qué momento se evalúa cada uno.
**Qué tenés que entender:** son mecanismos distintos que se resuelven en momentos distintos. **Es la causa número uno de errores incomprensibles en pipelines.**
**Tiempo de lectura:** 45 min
**Cuándo volver:** cada vez que una variable aparezca vacía.

### martinfowler.com
**Tipo:** referencia canónica de arquitectura
**URL:** https://martinfowler.com
**Qué cubre:** BFF, integración continua, refactoring, patrones de integración empresarial.
**Qué tenés que entender:** es la fuente del vocabulario que usa toda la industria. Cuando decís "BFF" en una reunión, viene de acá.
**Cuándo volver:** cuando necesites el nombre canónico de un patrón que estás describiendo con las manos.

### microservices.io
**Tipo:** catálogo de patrones (Chris Richardson)
**URL:** https://microservices.io
**Qué cubre:** descomposición de servicios, API Gateway, Saga, Database per Service.
**Cuándo volver:** al discutir fronteras entre servicios y quién es dueño de qué dato.

### Google SRE Book
**Tipo:** libro oficial, gratuito online
**URL:** https://sre.google/books/
**Qué cubre:** SLO, error budgets, monitoreo, respuesta a incidentes, postmortems.
**Qué tenés que entender:** el capítulo de SLO es directamente aplicable a datos (frescura, completitud). Es la mejor fuente gratuita para el concepto `data-governance-cost`.
**Tiempo de lectura:** variable — arrancá por SLO y por monitoreo
**Cuándo volver:** al definir compromisos de servicio sobre tus datos.

## Formatos y almacenamiento

### Apache Parquet — File Format
**Tipo:** spec oficial
**URL:** https://parquet.apache.org/docs/file-format/
**Qué cubre:** row groups, column chunks, pages, footer con estadísticas.
**Qué tenés que entender:** de dónde sale el predicate pushdown y por qué los archivos chicos son veneno.
**Tiempo de lectura:** 1 hora
**Cuándo volver:** cuando "columnar" te suene a palabra y no a mecanismo.

### Apache Iceberg — Spec
**Tipo:** spec oficial
**URL:** https://iceberg.apache.org/spec/
**Qué cubre:** metadata files, manifests, snapshots, evolución de esquema y de partición.
**Qué tenés que entender:** cómo la capa de metadata da atomicidad y time travel sobre archivos inmutables.
**Tiempo de lectura:** 2 horas (denso)
**Cuándo volver:** al evaluar lakehouse o interoperabilidad entre motores.

## APIs

### Google API Design Guide
**Tipo:** guía oficial
**URL:** https://cloud.google.com/apis/design
**Qué cubre:** recursos, métodos estándar, nombres, errores, paginación, versionado.
**Qué tenés que entender:**
- Orientación a recursos y métodos estándar
- Paginación con tokens
- Modelo de errores consistente
**Tiempo de lectura:** 3 horas
**Cuándo volver:** **el mejor documento gratuito de diseño de APIs que existe.** Si leés uno solo, este.

### Microsoft REST API Guidelines
**Tipo:** guía oficial pública
**URL:** https://github.com/microsoft/api-guidelines
**Qué cubre:** versionado, operaciones de larga duración, paginación, filtrado, errores.
**Tiempo de lectura:** 2 horas
**Cuándo volver:** al definir la política de versionado de una API propia.

### Zalando RESTful API Guidelines
**Tipo:** guía pública
**URL:** https://opensource.zalando.com/restful-api-guidelines/
**Qué cubre:** cientos de reglas con su razonamiento explícito.
**Qué tenés que entender:** el valor está en el *por qué* de cada regla, no en la regla.
**Cuándo volver:** cuando necesites justificar una decisión de diseño de API con argumentos.

### RFC 9110 — HTTP Semantics
**Tipo:** RFC (fuente autoritativa)
**URL:** https://www.rfc-editor.org/rfc/rfc9110.html
**Qué cubre:** qué significan realmente los métodos y los códigos de estado.
**Qué tenés que entender:** qué métodos son seguros, cuáles idempotentes, y el significado exacto de cada familia de códigos.
**Cuándo volver:** cuando haya discusión sobre qué código devolver.

### RFC 9457 — Problem Details for HTTP APIs
**Tipo:** RFC
**URL:** https://www.rfc-editor.org/rfc/rfc9457.html
**Qué cubre:** el formato estándar de cuerpo de error. (Obsoleta a la RFC 7807.)
**Cuándo volver:** al diseñar el modelo de errores de una API.

### OpenAPI Specification
**Tipo:** spec
**URL:** https://spec.openapis.org/oas/latest.html
**Qué cubre:** la estructura formal del contrato.
**Cuándo volver:** al escribir o leer una spec.

## MCP

### MCP Specification
**Tipo:** spec oficial
**URL:** https://modelcontextprotocol.io/specification
**Qué cubre:** primitivas (tools, resources, prompts), transportes, negociación de capacidades, modelo de seguridad.
**Qué tenés que entender:**
- La distinción tools / resources / prompts y sus modelos de permiso
- Arquitectura cliente-servidor
- Implicaciones de seguridad, especialmente en servidores locales
**Tiempo de lectura:** 2 horas (completa) o 30 min (overview)
**Cuándo volver:** antes de escribir un servidor MCP.

### MCP Reference Servers
**Tipo:** repos oficiales
**URL:** https://github.com/modelcontextprotocol/servers
**Qué cubre:** implementaciones de referencia en Python y TypeScript.
**Qué tenés que entender:** patrones de exposición de resources, convenciones de nombres de tools, manejo de errores.
**Cuándo volver:** como plantilla para tu propio servidor.

---

# CAPA SINTAXIS

Referencia rápida, para consultar con el editor abierto. **No sirve para aprender el concepto** — sirve para
escribirlo bien una vez que lo entendiste.

## Snowflake
| Doc | URL | Resuelve |
|---|---|---|
| SQL Command Reference | https://docs.snowflake.com/en/sql-reference-commands | DDL/DML exacto: `CREATE WAREHOUSE`, `COPY INTO`, `MERGE`, `CREATE DYNAMIC TABLE` |
| Function Reference | https://docs.snowflake.com/en/sql-reference-functions | Semi-estructurado (`FLATTEN`, `PARSE_JSON`, path `:`), ventanas, fechas |
| Snowpark Python API | https://docs.snowflake.com/en/developer-guide/snowpark/reference/python/index.html | API de DataFrame si salís de SQL puro |
| Snowflake CLI | https://docs.snowflake.com/en/developer-guide/snowflake-cli/index | Lo que vas a usar en CI |
| `ACCOUNT_USAGE` schema | https://docs.snowflake.com/en/sql-reference/account-usage | Vistas de consumo, query history, metering |

## dbt
| Doc | URL | Resuelve |
|---|---|---|
| dbt Reference | https://docs.getdbt.com/reference/references-overview | `dbt_project.yml`, configs de modelo, properties YAML |
| Jinja functions | https://docs.getdbt.com/reference/dbt-jinja-functions | `ref`, `source`, `config`, `var`, `this`, `run_query` |
| dbt Commands | https://docs.getdbt.com/reference/dbt-commands | `build` vs `run`, flags |
| Node selection syntax | https://docs.getdbt.com/reference/node-selection/syntax | Selectores: `+model`, `tag:`, `state:modified`, `--defer` |
| Jinja (oficial) | https://jinja.palletsprojects.com/ | Cuando el macro se pone raro y no es culpa de dbt |
| dbt Package Hub | https://hub.getdbt.com/ | Packages disponibles |
| dbt-utils | https://github.com/dbt-labs/dbt-utils | Macros y tests ya resueltos |

## Airflow
| Doc | URL | Resuelve |
|---|---|---|
| Python API Reference | https://airflow.apache.org/docs/apache-airflow/stable/python-api-ref.html | Firmas exactas de operators, hooks, decoradores |
| Templates Reference | https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html | Variables templadas — **fuente #1 de bugs de backfill** |
| Configuration Reference | https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html | Toda la configuración: paralelismo, pools, concurrencia |
| Astronomer Registry | https://registry.astronomer.io/ | Buscador de providers y operators |

## Azure DevOps
| Doc | URL | Resuelve |
|---|---|---|
| YAML Schema Reference | https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/ | Cada clave válida |
| Predefined Variables | https://learn.microsoft.com/azure/devops/pipelines/build/variables | Variables del sistema disponibles |
| Expressions | https://learn.microsoft.com/azure/devops/pipelines/process/expressions | Los momentos de evaluación |
| Tasks Reference | https://learn.microsoft.com/azure/devops/pipelines/tasks/reference/ | Catálogo de tasks integradas |
| Variable groups | https://learn.microsoft.com/azure/devops/pipelines/library/variable-groups | Configuración y enlace a Key Vault |

## APIs / MCP / otros
| Doc | URL | Resuelve |
|---|---|---|
| JSON Schema | https://json-schema.org/understanding-json-schema/ | Base de OpenAPI **y** de los inputs de tools MCP |
| IANA HTTP Status Codes | https://www.iana.org/assignments/http-status-codes/ | Registro autoritativo de códigos |
| MCP SDKs | https://github.com/modelcontextprotocol | Firmas reales de tools, resources, transports |
| PostgreSQL SQL docs | https://www.postgresql.org/docs/current/sql.html | Semántica SQL de referencia cuando Snowflake es ambiguo |

---

# Gap de libros (📕 PENDIENTE)

**Estado**: los tres libros canónicos del dominio NO están cargados en esta skill. Los conceptos marcados
`📕 pendiente` en `concepts.md` se enseñan hoy con las fuentes de arriba más conocimiento no verificado.

**Obligación al enseñar esos conceptos**: declarar explícitamente qué parte no está verificada.
*"Esta parte te la doy de memoria y no la pude contrastar con la fuente — tomala con pinzas."*

| Libro | Qué aporta que no está en ninguna fuente gratuita | Sustituto usado hoy |
|---|---|---|
| *Fundamentals of Data Engineering* — Reis & Housley | El lifecycle completo con sus corrientes de fondo; el marco mental del oficio | Azure Data Guide + este archivo |
| *Designing Data-Intensive Applications* — Kleppmann | Storage engines, replicación, particionamiento, consistencia | Paper de Snowflake + spec de Parquet + SRE Book |
| *The Data Warehouse Toolkit* — Kimball | Modelado dimensional: proceso de diseño, tipos de hechos, SCD completas | Guía de estructura de dbt + *Design Tips* públicos del Kimball Group |

**Para cerrar el gap**: si el usuario aporta los PDF/epub, se leen y se destilan los frameworks
**parafraseados** en los milestones correspondientes (nunca texto literal), y se saca la marca `📕 pendiente`
de `concepts.md`.

---

# 🌐 Comunidades (capa wisdom)

El conocimiento (los docs) y los ejercicios (tu laburo) no alcanzan: la sabiduría viene de contrastar con
gente que te discute el número. Cuando un concepto llega a `mastered` o terminás un `project`, andá a que te
lo rompan.

**Regla**: elegí DOS, no diez. Una general y una de lo que estás laburando AHORA.

## Generales

### dbt Community (Slack + Discourse)
**Dónde:** Slack de la comunidad + foro Discourse
**Link:** https://www.getdbt.com/community/ (desde ahí el acceso a Slack) — foro: https://discourse.getdbt.com/
**Para qué sirve:** modelado, estructura de proyectos, incrementales, testing. Es donde nació el rol de Analytics Engineer.
**Reputación:** maintainers activos y practitioners con proyectos grandes. Los threads viejos del foro son oro.
**Cómo aportar:** documentá una estrategia de incrementales con los números de tu caso, o respondé a alguien peleando con algo que ya resolviste.

### Locally Optimistic
**Dónde:** comunidad y blog de líderes de equipos de datos
**Link:** https://locallyoptimistic.com/
**Para qué sirve:** la parte organizacional del oficio — ownership, cómo se estructura un equipo de datos, cómo se negocia con el negocio. Cubre justo lo que la doc técnica no.
**Reputación:** gente que dirige equipos de datos reales. Nivel alto en criterio, no en sintaxis.
**Cómo aportar:** escribí sobre un problema organizacional que resolviste, no técnico.

### r/dataengineering
**Dónde:** Reddit
**Link:** https://www.reddit.com/r/dataengineering/
**Para qué sirve:** transversal — herramientas, arquitectura, mercado laboral, quejas honestas sobre lo que no funciona.
**Reputación:** mezcla de niveles. Los hilos de arquitectura suelen tener buen contenido; los de "qué herramienta uso" son ruido.
**Cómo aportar:** compartí un postmortem de algo que se rompió en producción. Eso es lo más valioso y lo más escaso.

### Hacker News
**Dónde:** foro (Y Combinator)
**Link:** https://news.ycombinator.com/
**Para qué sirve:** calibrar señal vs hype sobre cualquier herramienta nueva. Los comentarios suelen valer más que el artículo.
**Cómo aportar:** comentá con experiencia técnica real. Ahí la opinión vacía se nota.

## Por hito

### Hito 2 — Snowflake
**Snowflake Community Forums** — https://community.snowflake.com/ — para dudas de comportamiento del motor y de costo. Empleados de Snowflake participan.
**Snowflake Docs Release Notes** — en docs.snowflake.com — no es comunidad, pero es la única forma de enterarte de features nuevas que cambian decisiones de arquitectura.

### Hito 3 — dbt
**dbt Slack** (ver arriba) — los canales de modelado y de incrementales son los que más devuelven.

### Hito 4 — Airflow
**Apache Airflow Slack** — buscar el invite desde https://airflow.apache.org/community/ — committers presentes.
**Airflow Summit** — buscar el canal oficial en YouTube — charlas de casos de producción reales, no demos.

### Hito 5 — APIs & MCP
**MCP Community (GitHub Discussions)** — https://github.com/modelcontextprotocol — ahí está la gente que define el protocolo.
**OpenAPI Initiative** — https://www.openapis.org/ — para discusiones de spec.

### Hito 6 — System Design & Delivery
**MLOps Community** — https://mlops.community/ — Slack grande y serio; cubre producción, monitoreo y compliance práctico, con overlap fuerte con data engineering.

## LATAM / Argentina

> Los links exactos de meetups rotan y muchos viven en plataformas distintas. NO se inventan URLs: acá van
> pistas de búsqueda, verificá vos cuál está vivo hoy.

**Comunidades de datos de Argentina** — buscar en meetup.com: "data engineering Buenos Aires", "Machine Learning Argentina". Verificá la fecha del último evento antes de sumarte.
**Python Argentina (PyAr)** — https://www.python.org.ar/ — comunidad veterana y respetada; base sólida de Python, que es el stack de casi todo lo de datos.
**Comunidades de datos en español** — buscar referentes de datos LATAM en LinkedIn y desde ahí sus comunidades. Hay hambre de buen contenido técnico en castellano: explicar bien un concepto difícil en español te posiciona rápido.

## Cómo usar esto (postura del mentor)

- **No te sumes a diez comunidades.** Te perdés y no aportás en ninguna. Dos: una general y una del hito que estás laburando ahora.
- **Wisdom = aportar, no solo leer.** Lurkear da la ilusión de aprender. Posteá tu decisión con los números, bancá que te la discutan.
- **El trigger es claro**: cuando el mentor te marca un concepto como `mastered`, o cuando terminás un `project`. Ese es el momento. No antes (no tenés nada que mostrar), no después (perdés el momentum).
- **En data engineering, el aporte más valioso es el postmortem.** Todo el mundo cuenta lo que construyó; casi nadie cuenta lo que se le rompió y por qué. Eso es lo que la comunidad necesita y lo que más te posiciona.
