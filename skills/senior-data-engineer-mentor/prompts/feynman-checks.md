# Feynman Checks — rúbricas para pasar a `mastered`

**Cuándo se carga**: solo cuando un concepto está en `practiced` y hay evidencia de una decisión de diseño
tomada por el usuario. El Feynman check es el ÚLTIMO paso antes de `mastered`, nunca el primero.

**Por qué existe**: explicar algo con jerga es fácil — la jerga tapa los huecos. Explicarlo sin la jerga
obliga a tener el modelo mental completo. Por eso cada check **prohíbe una palabra**: justamente la que
permite esconder que no se entiende.

---

## Requisito previo (no negociable)

Antes de correr un Feynman check, verificá las DOS condiciones:

1. El concepto está en `practiced` (ya hizo el ejercicio de *Gimnasio* de `concepts.md` y volvió con un resultado real).
2. Hay una **decisión de diseño** tomada y defendida con números: eligió una materialización y justificó el costo, decidió poner o sacar una clustering key con datos, definió una estrategia de paginación, etc.

Si falta alguna, NO corras el check. Decilo: *"Todavía no. Te falta {la condición}. El Feynman es el
último paso, no el atajo."*

---

## Rúbrica general (aplica a todos)

Una explicación aprueba cuando cumple las cuatro:

| Criterio | Aprueba si... | Falla si... |
|---|---|---|
| **Sin la palabra prohibida** | Explica el mecanismo con otras palabras | Necesita la palabra o da una vuelta que la reintroduce |
| **Causal, no descriptivo** | Explica POR QUÉ funciona así | Describe QUÉ hace sin el mecanismo |
| **Con la consecuencia práctica** | Deriva qué cambia en su trabajo | Se queda en la teoría |
| **Con el límite** | Dice cuándo NO aplica o cuándo se rompe | Lo presenta como universalmente bueno |

**Veredicto**: 4/4 → `mastered`. 3/4 → queda en `practiced` con nota de qué faltó. ≤2/4 → `practiced` y se
identifica el hueco concreto para volver a enseñarlo.

**Regla de honestidad**: si la explicación es fluida pero vacía, decilo. *"Sonó bien pero no explicaste el
mecanismo, solo lo renombraste."* Aprobar por simpatía destruye el valor de todo el sistema de niveles.

---

## Checks por concepto

### Hito 1 — Fundamentos

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `de-lifecycle` | Explicale a alguien de negocio qué hace un data engineer y dónde empieza y termina su responsabilidad | "pipeline" | Que hay etapas con dueños distintos y que el trabajo es gestionar los contratos entre ellas |
| `dimensional-modeling` | Explicale a alguien de negocio qué representa una fila de tu tabla de hechos y por qué eso importa | "grano" | Que si una fila significa dos cosas distintas, las sumas se duplican |
| `columnar-storage` | Explicá por qué pedir tres columnas cuesta menos que pedir todas | "columnar" | Que los datos se guardan agrupados por campo, así que se lee del disco solo lo pedido |
| `batch-vs-streaming` | Explicale a un gerente por qué su pedido de "tiempo real" puede costar cinco veces más | "streaming" | Que el costo sube con la frecuencia y que la pregunta real es qué decisión se toma y cuándo |
| `idempotency-backfill` | Explicá por qué usar la fecha de hoy adentro de una transformación es un bug | "idempotente" | Que al reprocesar el pasado, "hoy" ya es otro día, y el resultado cambia sin dar error |
| `table-formats` | Explicale a alguien que un directorio con archivos no es una tabla | "ACID" | Que hace falta algo que diga qué archivos forman la tabla en cada momento, o se ven escrituras a medias |

### Hito 2 — Snowflake

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `snowflake-architecture` | Explicá por qué apagar el compute no borra los datos y por qué dos equipos no se pisan | "arquitectura" | Que el lugar donde viven los datos y el lugar donde se procesan son cosas separadas |
| `micro-partitions` | Explicá cómo Snowflake evita leer una tabla entera al filtrar | "índice" y "partición" | Que los datos están en bloques que traen anotado su rango de valores, y se descartan bloques enteros |
| `virtual-warehouses` | Explicale a finanzas por qué un warehouse más grande puede costar lo mismo | "escalar" | Que se paga por tiempo encendido, así que el doble de recursos en la mitad del tiempo es el mismo gasto |
| `snowflake-caching` | Explicá por qué medir una optimización con el cronómetro te puede mentir | "caché" | Que el resultado puede venir guardado de una corrida anterior, así que el tiempo no refleja el trabajo real |
| `clustering-pruning` | Explicá por qué no le ponés clustering key a todas las tablas | "índice" | Que reordenar los datos se cobra de forma continua, y solo se paga si de verdad se descartan más bloques |
| `snowflake-cost` | Explicale a tu equipo por dónde arrancar a bajar la factura | "optimizar" | Que primero hay que saber quién gastó, y para eso hay que separar el compute por área |

### Hito 3 — dbt

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `dbt-project-structure` | Explicale a alguien nuevo por qué no puede meter lógica de negocio en la primera capa | "staging" | Que esa capa es la frontera con la fuente, y si le metés reglas, cualquiera que la use hereda decisiones que no pidió |
| `dbt-ref-lineage` | Explicá por qué escribir el nombre de la tabla a mano rompe cosas que no se ven | "DAG" | Que la herramienta deduce el orden y el ambiente de esas referencias; sin ellas, adivina |
| `materializations` | Explicá cómo decidís si un modelo se guarda o se recalcula | "materialización" | Que se compara lo que cuesta construirlo contra cuánto se consulta |
| `incremental-models` | Explicá por qué un modelo que solo procesa lo nuevo puede perder datos sin avisar | "incremental" | Que si un registro llega con fecha vieja después de que el proceso ya avanzó, nunca entra, y no hay error |
| `dbt-tests-contracts` | Explicale a un consumidor qué le garantiza tu mart y qué no | "test" | Que se verifican propiedades que alguien pensó en verificar, no la corrección del negocio |
| `dbt-jinja-macros` | Explicá por qué no podés decidir un filtro según los datos del modelo que estás construyendo | "compilación" | Que el texto de la consulta se arma antes de que exista cualquier resultado |

### Hito 4 — Airflow

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `airflow-architecture` | Explicá por qué no hay que poner una llamada a una API afuera de las tareas | "parseo" | Que el archivo se lee una y otra vez para descubrir cambios, aunque nada esté corriendo |
| `dag-scheduling` | Explicale a alguien que Airflow no es un cron | "cron" | Que cada corrida es dueña de una ventana de datos, y se dispara cuando esa ventana cierra |
| `operators-hooks-taskflow` | Explicá qué puede viajar entre tareas y qué no | "XCom" | Que lo que se pasa se guarda en la base interna del orquestador, así que van referencias, no datos |
| `dag-idempotency` | Explicá por qué la fecha tiene que venir de afuera de la tarea | "template" | Que si la tarea decide sola qué día es, reejecutarla en otro momento procesa otra cosa |
| `airflow-assets` | Explicá por qué encadenar por horario es peligroso | "dataset" | Que el horario es una suposición sobre cuánto tarda lo anterior, y cuando falla no da error |
| `airflow-scaling` | Explicá por qué agregar workers a veces no arregla nada | "concurrencia" | Que puede haber lugares ocupados por tareas que solo esperan, así que no falta capacidad, falta lugar |

### Hito 5 — APIs & MCP

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `rest-resource-design` | Explicá por qué el código de estado importa aunque tu cliente lea el cuerpo | "REST" | Que hay infraestructura en el medio que solo mira ese código para decidir si reintentar o alertar |
| `api-contracts` | Explicale a un consumidor qué podés cambiar sin avisarle y qué no | "breaking change" | Que agregar suele ser seguro y sacar o cambiar el tipo no, y que hay cambios invisibles peores que los visibles |
| `api-pagination-filtering` | Explicá por qué leer "de a 100" puede perder registros | "paginación" | Que si se cuenta desde el principio cada vez y entran filas nuevas, todo se corre |
| `api-auth` | Explicá por qué el secreto no puede estar en el código | "Key Vault" | Que el historial del repositorio es permanente y que quien lo lee obtiene el acceso |
| `api-reliability` | Explicá por qué reintentar todo empeora las cosas | "backoff" | Que algunos errores no cambian por repetirlos, y que reintentar todos a la vez tira lo que se estaba recuperando |
| `mcp-protocol` | Explicale a alguien de backend qué es MCP sin mencionar modelos de lenguaje | "LLM" y "agente" | Que es un contrato estándar para exponer capacidades, escrito una vez y consumido por cualquier cliente compatible |

### Hito 6 — System Design & Delivery

| Concepto | Consigna | Palabra prohibida | Lo que tiene que aparecer |
|---|---|---|---|
| `backend-frontend-split` | Explicale a producto por qué el dashboard no consulta el warehouse directo | "seguridad" | Que las credenciales quedan del lado del usuario y que el costo crece con cada persona que abre la pantalla |
| `data-serving-layer` | Explicá por qué tres áreas reportan tres números distintos de lo mismo | "semantic layer" | Que cada una escribió su propia definición, así que el problema es de propiedad antes que técnico |
| `azure-pipelines` | Explicá por qué una variable puede aparecer vacía aunque esté definida | "expresión" | Que hay cosas que se resuelven al armar el pipeline y otras cuando corre, y no ven lo mismo |
| `cicd-for-data` | Explicá por qué el CI de datos no puede copiar al de software | "state:modified" | Que acá cada verificación construye tablas reales, así que verificar todo cuesta plata de verdad |
| `iac-secrets` | Explicá por qué borrar una contraseña de un commit no alcanza | "historial" | Que ya quedó guardada y copiada, así que lo único que la desactiva es cambiarla |
| `data-governance-cost` | Explicá por qué el lineage no te dice todo el impacto de un cambio | "lineage" | Que solo ve lo que pasa por la herramienta, y afuera hay gente consultando directo |

---

## Protocolo de aplicación

1. Verificá el requisito previo (nivel `practiced` + decisión de diseño defendida). Si falta, no corras el check.
2. Presentá la consigna con la palabra prohibida bien clara: *"Explicame {X} sin usar la palabra `{Y}`. Si la usás, arrancamos de nuevo."*
3. Esperá la explicación completa. NO ayudes, NO completes la frase.
4. Evaluá con la rúbrica general (4 criterios) y verificá que aparezca lo de la columna "Lo que tiene que aparecer".
5. Devolvé el veredicto con el detalle de qué criterio falló, si falló alguno.
6. **Persistí**: `mem_save` sobre `skill/data-engineer-mentor/mastery/{slug}` con el nuevo nivel, la evidencia (`"feynman check aprobado {fecha} + decisión: {cuál}"`), `next_review` recalculado con el intervalo de `mastered`, e `history[]` actualizado.
7. Si aprobó → aplicá la regla de **wisdom** del SKILL.md: empujalo a defender esa decisión fuera del entorno de aprendizaje (ADR interno, presentación al equipo, writeup público).

## Anti-patterns del check (NO hacer)

- **NO** correr el check apenas terminás de explicar el concepto. Eso mide fluidez, no retención.
- **NO** aceptar un sinónimo de la palabra prohibida. Si prohibiste "índice" y dice "estructura de búsqueda auxiliar", es la misma palabra con más letras.
- **NO** aprobar una explicación fluida pero vacía. La fluidez es exactamente lo que este check está diseñado para atravesar.
- **NO** dar pistas ni completar la idea. El silencio incómodo es parte del método.
- **NO** subir a `mastered` sin la decisión de diseño previa, por buena que sea la explicación. Explicar no es haber decidido.
