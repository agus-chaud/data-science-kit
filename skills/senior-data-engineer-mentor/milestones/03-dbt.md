# Hito 3 — dbt

> **Chequeo de versión obligatorio antes de enseñar**: dbt movió sintaxis entre versiones (selectores,
> materializaciones nuevas como `materialized_view` y las dynamic tables de Snowflake, `dbt build` vs
> `dbt run`, unit tests). Preguntá o verificá qué versión usa el usuario ANTES de afirmar comportamiento.

## Por qué importa (perspectiva corporativa)

dbt es la herramienta que más rápido se aprende a usar y más difícil se aprende a usar BIEN. En dos horas
escribís tu primer modelo. En dos años seguís descubriendo por qué el proyecto se volvió inmanejable.

Y acá está el punto que casi nadie dice: **dbt no es una herramienta de SQL, es una herramienta de
ingeniería de software aplicada a SQL**. Lo que trajo dbt no fue el `SELECT` — el `SELECT` ya existía. Trajo
control de versiones, dependencias explícitas, tests, ambientes, documentación y CI para gente que antes
escribía procedimientos almacenados sin ninguna de esas cosas. Si usás dbt como "un lugar donde guardo mis
queries", estás usando el 10% de la herramienta y pagando el 100% de su complejidad.

En el mercado: **Analytics Engineer** es hoy un rol propio y dbt es su herramienta central. Y la
diferenciación senior no está en escribir modelos — está en el diseño del proyecto: capas, contratos,
estrategia de incrementales, CI que no cueste una fortuna. Eso es lo que se pregunta en una entrevista de
verdad, y es exactamente lo que no aparece en los tutoriales.

## Conceptos de este hito

### dbt-project-structure

**Qué es**: La organización en tres capas: **staging** (una vista 1:1 por tabla fuente, renombra y castea,
sin lógica de negocio), **intermediate** (pasos de transformación reutilizables, no se exponen al consumo)
y **marts** (los modelos de consumo, orientados al negocio, con grano declarado).

**La trampa del junior**: una carpeta `models/` plana con 200 archivos y nombres tipo `ventas_final_v2`.
Nadie sabe cuál es autoritativo, la lógica de negocio está duplicada en seis modelos, y cambiar el nombre
de una columna en la fuente rompe cosas en lugares impredecibles.

**Cómo lo piensa un senior**: cada capa tiene **una sola responsabilidad**, y esa restricción es lo que
mantiene el proyecto navegable a los tres años. Staging es la frontera con el mundo exterior: si la fuente
cambia un nombre de columna, se arregla en UN archivo. Intermediate existe para no repetir lógica.
Marts es lo único que consume el negocio. La regla que se deriva de esto y que ordena todo: **cada modelo
de staging referencia una y solo una fuente, y ningún mart referencia una fuente directamente**.

**Tradeoffs reales**:

| Decisión | Opción | Cuándo | Contra |
|---|---|---|---|
| Capa staging | Vistas | Default: liviano, siempre fresco | Se recomputa en cada lectura |
| Capa staging | Tablas | Fuente muy pesada y muchos consumidores | Storage + orquestación extra |
| Intermediate | Ephemeral (CTE inyectada) | Paso lógico sin valor de consulta propia | No debuggeable directo, infla el SQL final |
| Intermediate | Vista o tabla | Cuando querés inspeccionarlo o lo usan varios | Un objeto más en el warehouse |
| Marts | Por dominio de negocio (`finance/`, `marketing/`) | Default en equipos medianos y grandes | Requiere disciplina de ownership |
| Marts | Todo junto | Equipos muy chicos | Escala mal apenas crece el equipo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Para qué sirve la capa de staging si es 1:1 con la fuente?*
  A: Para aislar el cambio. Es la única capa que conoce los nombres y tipos originales; ahí se renombra, se castea y se normaliza. Si la fuente cambia, se arregla en un archivo en vez de en cincuenta modelos. Además da un vocabulario consistente para todo el proyecto.
- Q (senior): *Tenés un proyecto dbt heredado con 300 modelos en una carpeta plana. ¿Por dónde empezás?*
  A: No refactorizo todo de una. Primero mapeo el DAG y busco los modelos hoja que realmente consume el negocio (los que están en dashboards): ese es el subconjunto que importa. Después, para cada fuente, creo su modelo de staging y voy migrando referencias directas de a poco. Los modelos huérfanos que nadie consulta los identifico con el log de queries del warehouse y los deprecio. El objetivo es reducir el grafo antes que reordenarlo.
- Q (trampa): *¿Conviene poner lógica de negocio en staging para no repetirla?*
  A: No. Staging tiene que ser predecible y aburrido: renombrar, castear, limpiar tipos. En cuanto le metés lógica de negocio, dejás de tener una frontera limpia con la fuente y cualquier consumidor de staging hereda decisiones que no pidió. La lógica compartida va en intermediate, que existe justamente para eso.

### dbt-ref-lineage

**Qué es**: `ref('modelo')` y `source('schema','tabla')` son las funciones que construyen el **DAG**. dbt
resuelve el orden de ejecución, el lineage y el nombre físico del objeto según el ambiente a partir de ellas.

**La trampa del junior**: escribir `FROM analytics.dim_cliente` hardcodeado porque "es más claro". Con eso
rompe todo: dbt no sabe que ese modelo es una dependencia, así que puede correrlo en el orden equivocado; el
lineage queda incompleto; y en el ambiente de dev sigue apuntando a producción — que es cómo se termina
escribiendo en prod desde una rama.

**Cómo lo piensa un senior**: `ref()` no es azúcar sintáctica, es **el mecanismo por el que dbt existe**.
Es lo que convierte un conjunto de queries en un grafo con dependencias declaradas, y de ahí salen el orden,
el lineage, la separación de ambientes, el `--defer` y el `state:modified` del CI. Cada referencia
hardcodeada es un nodo que se cae del grafo — y el grafo es el producto.

**Tradeoffs reales**:

| Patrón | Efecto |
|---|---|
| `ref('modelo')` | ✅ Dependencia en el DAG, resolución por ambiente, lineage completo |
| `source('sch','tabla')` | ✅ Marca la frontera del proyecto, habilita freshness tests |
| `schema.tabla` hardcodeado | ❌ Nodo perdido, orden impredecible, apunta a prod desde dev |
| `ref()` cruzando proyectos (dbt Mesh) | Permite dividir proyectos grandes; requiere contratos y versiones |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué diferencia hay entre `ref` y `source`?*
  A: `ref` apunta a otro modelo del proyecto y crea una dependencia interna en el DAG. `source` apunta a una tabla externa que dbt no construye, marca la frontera de entrada del proyecto y habilita tests de frescura sobre la fuente.
- Q (senior): *¿Por qué `ref()` es lo que hace posible el CI barato de dbt?*
  A: Porque el DAG explícito permite calcular qué cambió y qué depende de lo que cambió. Con eso el CI corre solo los modelos modificados y sus descendientes (`state:modified+`) en vez del proyecto completo, y con `--defer` toma el resto desde producción sin reconstruirlo. Sin `ref()` no hay grafo, y sin grafo la única opción es correr todo.
- Q (trampa): *Un modelo referencia una tabla que existe en el warehouse pero no es un modelo dbt ni está declarada como source. Funciona. ¿Está mal?*
  A: Funciona hoy y falla mañana. dbt no sabe que esa dependencia existe, así que puede ejecutar el modelo antes de que la tabla esté actualizada, el lineage miente, y en un ambiente distinto el objeto puede no existir. Hay que declararla como `source` aunque sea "solo una tabla más".

### materializations

**Qué es**: Cómo dbt convierte tu `SELECT` en un objeto del warehouse. Las principales: **view** (no
almacena, se recalcula al leer), **table** (recrea la tabla completa cada corrida), **incremental**
(inserta/actualiza solo lo nuevo), **ephemeral** (se inyecta como CTE, no crea objeto), y las
materializaciones gestionadas por el warehouse (materialized views, dynamic tables en Snowflake).

**La trampa del junior**: dejar todo en `table` porque "es más rápido de consultar". A los seis meses el
proyecto tarda tres horas en correr y quema créditos recreando tablas de miles de millones de filas donde
cambian mil.

**Cómo lo piensa un senior**: la materialización es **una decisión de costo, no de estilo**. La pregunta
que la ordena: *¿cuánto cuesta reconstruirlo vs cuánto se consulta?*. Modelo chico y muy consultado → tabla.
Modelo chico y poco consultado → vista. Modelo grande que crece por append → incremental. Paso intermedio que
nadie consulta → ephemeral. Y una regla que ahorra dolor: **empezá con vista o tabla, pasá a incremental
solo cuando el costo lo justifique** — el incremental agrega complejidad real (datos que llegan tarde,
reprocesos, divergencia con full-refresh) y no se paga sola en tablas chicas.

**Tradeoffs reales**:

| Materialización | Costo de build | Costo de consulta | Cuándo |
|---|---|---|---|
| `view` | Nulo | Alto (recalcula siempre) | Staging, modelos poco consultados |
| `table` | Alto (recrea todo) | Bajo | Marts chicos/medianos muy consultados |
| `incremental` | Bajo tras la primera | Bajo | Tablas grandes append-only o con updates acotados |
| `ephemeral` | Nulo | Se hereda al modelo padre | Pasos intermedios sin consumo directo |
| `materialized_view` / dynamic table | Gestionado por el warehouse (serverless) | Muy bajo | Frescura casi continua sin orquestar; ojo con el costo permanente |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cuándo usarías `ephemeral`?*
  A: Para un paso lógico intermedio que ningún consumidor va a consultar directamente y que se usa en pocos modelos. Se inyecta como CTE en el modelo que lo referencia, así que no crea objeto ni cuesta storage. La contra es que no lo podés inspeccionar en el warehouse y, si lo usan muchos modelos, inflás el SQL de todos.
- Q (senior): *¿Cómo decidís si un modelo pasa de `table` a `incremental`?*
  A: Comparo el costo de reconstrucción completa contra el volumen que realmente cambia. Si recrear cuesta mucho y el delta diario es una fracción chica, el incremental se paga. Pero antes verifico dos cosas: que exista una clave confiable para el merge, y qué tan tarde pueden llegar los datos — porque eso define la ventana de reproceso (lookback) y es lo que hace fallar a los incrementales mal diseñados.
- Q (trampa): *Las dynamic tables / materialized views se actualizan solas. ¿Entonces son siempre mejores?*
  A: Se actualizan solas y por eso consumen compute serverless de forma continua, tengan o no consumidores. Son excelentes cuando necesitás frescura sin orquestar, pero en un modelo que se consulta dos veces por día son un gasto permanente para un beneficio ocasional. La decisión es la frecuencia de consumo contra la frecuencia de refresco, igual que siempre.

### incremental-models

**Qué es**: Modelos que solo procesan filas nuevas o modificadas. El macro `is_incremental()` es verdadero
cuando el modelo ya existe, no se pidió `--full-refresh`, y está configurado como incremental. La estrategia
(`merge`, `delete+insert`, `append`, `insert_overwrite`, microbatch) define cómo se escriben esas filas.

**La trampa del junior**: filtrar por `WHERE fecha > (SELECT MAX(fecha) FROM {{ this }})` y darlo por
resuelto. Ese patrón pierde silenciosamente todo registro que llegue tarde: si un evento del martes entra el
jueves, el modelo ya avanzó el máximo al jueves y ese registro nunca entra. El error no rompe nada — solo
faltan datos, y nadie se entera hasta que el negocio reclama.

**Cómo lo piensa un senior**: un modelo incremental tiene **cuatro decisiones explícitas**, y omitir
cualquiera es un bug esperando: (1) la **clave única** real, (2) la **ventana de lookback** que cubre los
datos que llegan tarde, (3) qué pasa con **updates y deletes** en la fuente, y (4) con qué frecuencia se
hace un **full-refresh de reconciliación**. Y una regla no negociable: **el resultado del incremental tiene
que ser idéntico al del full-refresh**. Si divergen, el incremental está mal — y esa comparación se testea,
no se asume.

**Tradeoffs reales**:

| Estrategia | Maneja updates | Costo | Cuándo |
|---|---|---|---|
| `append` | ❌ No | Mínimo | Eventos inmutables puros, sin reprocesos |
| `merge` (default en Snowflake) | ✅ Sí | Medio-alto en tablas grandes | Default cuando hay clave única confiable |
| `delete+insert` | ✅ Por partición | Medio | Reemplazo de ventanas completas por fecha |
| `insert_overwrite` | ✅ Por partición | Medio | Lake/particionado, reemplazo de partición atómico |
| `microbatch` (versiones recientes) | ✅ Por lote temporal | Medio | Ventanas grandes procesadas en lotes independientes y reintentables |
| Full-refresh siempre | ✅ Trivialmente | Alto | Tablas chicas — y es la opción correcta más veces de lo que la gente cree |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué hace `is_incremental()`?*
  A: Devuelve verdadero cuando el modelo está configurado como incremental, ya existe físicamente y no se está corriendo con `--full-refresh`. Se usa para agregar el filtro que limita las filas procesadas solo en las corridas incrementales, dejando la primera corrida y los full-refresh completos.
- Q (senior): *Tu modelo incremental está perdiendo registros silenciosamente. ¿Qué mirás?*
  A: Primero comparo el incremental contra un full-refresh sobre el mismo período en un esquema paralelo: eso confirma y cuantifica la divergencia. Después reviso el filtro: si compara contra `MAX()` del destino, cualquier dato que llegue con fecha anterior queda afuera para siempre. La corrección es una ventana de lookback basada en la latencia real de la fuente (medida, no adivinada), o pasar a una estrategia por partición que reemplace ventanas completas. Y agrego un test de reconciliación periódico para que el problema no vuelva a ser invisible.
- Q (trampa): *¿El `unique_key` garantiza que no haya duplicados?*
  A: Garantiza que el merge deduplique **contra el destino**, no dentro del lote entrante. Si el batch trae dos filas con la misma clave, el merge puede fallar o quedarse con una arbitraria según el motor. Hay que deduplicar en el modelo antes del merge, con un criterio explícito de cuál gana — típicamente la más reciente por timestamp de la fuente.

### dbt-tests-contracts

**Qué es**: Tests **genéricos** (`unique`, `not_null`, `accepted_values`, `relationships`) declarados en
YAML, tests **singulares** (una query SQL que no debe devolver filas), **contracts** que fuerzan el esquema
y tipos de salida del modelo, y `dbt build` que ejecuta modelos y tests en orden del DAG deteniendo lo que
depende de algo roto.

**La trampa del junior**: cero tests, o tests decorativos (`not_null` en la clave primaria y nada más), y
correr `dbt run` en vez de `dbt build`. Con `dbt run` los modelos corren aunque sus dependencias tengan
datos rotos, así que el error se propaga hasta el dashboard.

**Cómo lo piensa un senior**: los tests son el **contrato observable** del modelo. Y hay una jerarquía de
valor clara: el test que más rinde es el que verifica el **grano** (unicidad sobre la clave del grano),
porque su violación es la que produce duplicación de métricas — el error más caro y más difícil de detectar
a ojo. Después vienen los tests de relación (integridad referencial contra dimensiones) y los de negocio
(un total que tiene que cerrar contra la fuente). Los `not_null` sueltos son los de menor valor y son los
que todo el mundo pone. Los **contracts** suman otra cosa distinta: convierten el esquema de salida en un
compromiso público, y por eso son la herramienta cuando otro equipo consume tu mart.

**Tradeoffs reales**:

| Test | Qué protege | Costo de correrlo |
|---|---|---|
| `unique` sobre la clave del grano | Duplicación de métricas — el error más caro | Medio (agrupa la tabla) |
| `not_null` | Nulos inesperados | Bajo |
| `relationships` | Integridad referencial hecho↔dimensión | Medio (join) |
| `accepted_values` | Categorías nuevas sin avisar | Bajo |
| Test singular de negocio | Que el total cierre contra la fuente | Variable, suele ser el más valioso |
| Freshness sobre sources | Que la fuente esté actualizada | Bajo |
| `contract: enforced` | Que no rompas a tus consumidores | Nulo en runtime, se valida al construir |
| Severidad `warn` vs `error` | Ruido vs bloqueo | — |

**En entrevista te van a preguntar**:
- Q (mid): *¿Diferencia entre `dbt run` y `dbt build`?*
  A: `dbt run` solo ejecuta modelos. `dbt build` ejecuta modelos, tests, seeds y snapshots respetando el DAG, y si un test falla detiene los modelos que dependen de ese nodo. Es la diferencia entre propagar datos rotos y frenarlos en el punto donde se rompieron.
- Q (senior): *Tenés presupuesto para 5 tests en un mart nuevo. ¿Cuáles ponés?*
  A: Uno de unicidad sobre la clave del grano, porque su violación duplica métricas y es lo más caro. Uno de frescura sobre la fuente principal, porque un mart actualizado con datos viejos es peor que uno que falla. Uno de integridad referencial contra la dimensión más usada. Y dos de negocio: que el total cierre contra la fuente y que no haya períodos faltantes. Los `not_null` sueltos los dejo para después: detectan poco y son los que todos ponen primero.
- Q (trampa): *Todos tus tests pasan. ¿El mart está bien?*
  A: Los tests verifican lo que alguien pensó en verificar. Pasan verdes cuando la lógica de negocio está mal implementada de forma consistente, cuando falta una porción entera de datos que ningún test cubre, o cuando el grano está mezclado pero no hay test de unicidad. Los tests son un piso de confianza, no una prueba de corrección.

### dbt-jinja-macros

**Qué es**: dbt compila Jinja **antes** de que exista SQL. `ref()`, `config()`, `var()`, `is_incremental()`
y los macros se resuelven en tiempo de compilación; el resultado es SQL plano que se manda al warehouse.
`run_query()` permite ejecutar contra el warehouse durante la compilación.

**La trampa del junior**: pensar que Jinja "corre" junto al SQL, y escribir macros que intentan tomar
decisiones a partir de datos de la tabla que se está construyendo. O abusar de macros hasta que nadie puede
leer un modelo sin desenrollar cuatro niveles de indirección.

**Cómo lo piensa un senior**: **hay dos tiempos y no se mezclan** — tiempo de compilación (Jinja, Python,
conoce el proyecto) y tiempo de ejecución (SQL, en el warehouse, conoce los datos). Todo lo que necesite
saber de los datos va en SQL; todo lo que necesite saber del proyecto va en Jinja. Y el debugger de un
senior es `dbt compile` + leer `target/compiled/`: ahí ves exactamente el SQL que se ejecutó, y el 90% de
los bugs "raros" de dbt se resuelven mirando eso. Sobre macros: la regla es la misma que en cualquier
abstracción — se extrae cuando el patrón se repitió tres veces, no antes, y nunca a costa de que el modelo
deje de leerse como SQL.

**Tradeoffs reales**:

| Herramienta | Cuándo | Contra |
|---|---|---|
| Macro propio | Patrón repetido 3+ veces | Indirección; el modelo deja de leerse solo |
| `var()` | Parametrizar ventanas, flags de ambiente | Si abusás, el comportamiento depende de invocación invisible |
| `run_query()` en compilación | Listas dinámicas (pivots por categoría) | Query extra en cada compilación, incluso en CI |
| Paquetes (`dbt_utils`, expectations) | Tests y macros ya resueltos y probados | Dependencia externa a versionar |
| SQL plano sin Jinja | Legibilidad máxima | Duplicación cuando el patrón se repite |

**En entrevista te van a preguntar**:
- Q (mid): *¿Cómo debuggeás un modelo dbt que genera SQL raro?*
  A: `dbt compile` y leo el SQL generado en `target/compiled/`. Ahí veo exactamente lo que se manda al warehouse, con todos los `ref` resueltos y los macros expandidos. Después ese SQL lo corro directo en el warehouse para aislar si el problema es de Jinja o de la lógica.
- Q (senior): *¿Cuál es el riesgo de usar `run_query()` en un macro?*
  A: Que ejecuta una consulta contra el warehouse en cada compilación del proyecto, no en cada corrida del modelo. Eso significa costo y latencia en `dbt compile`, en `dbt docs`, y en cada job de CI — incluso cuando no se va a materializar nada. Además introduce una dependencia de datos que el DAG no ve: si la tabla que consulta no está lista, la compilación produce un SQL distinto sin avisar.
- Q (trampa): *¿Podés usar Jinja para decidir el filtro según un valor de la tabla que estás construyendo?*
  A: No, porque Jinja se resuelve antes de que el SQL se ejecute; en ese momento el resultado del modelo no existe todavía. Podrías consultar la versión anterior con `run_query()` o `{{ this }}`, pero eso es otra cosa y trae los problemas de arriba. Una decisión que depende de los datos del propio modelo va en SQL, con un `CASE` o un join, no en Jinja.

## Lo que la doc oficial cubre bien acá

- **How we structure our dbt projects** (https://docs.getdbt.com/best-practices/how-we-structure/1-guide-overview) — la guía de capas, naming y organización. Es criterio puro, no sintaxis.
- **Materializations** (https://docs.getdbt.com/docs/build/materializations) y **Incremental models** (https://docs.getdbt.com/docs/build/incremental-models) — con las estrategias y sus casos.
- **Data tests** (https://docs.getdbt.com/docs/build/data-tests) y **Model contracts** (https://docs.getdbt.com/docs/collaborate/govern/model-contracts).
- **Jinja functions reference** (https://docs.getdbt.com/reference/dbt-jinja-functions) — `ref`, `source`, `this`, `var`, `run_query` con su semántica exacta.
- **Node selection syntax** (https://docs.getdbt.com/reference/node-selection/syntax) — selectores, `state:modified`, `--defer`. Es lo que después usás en el Hito 6 para el CI.
- **dbt Developer Blog** (https://docs.getdbt.com/blog) — artículos con el porqué detrás de las decisiones de diseño.

## Gaps

- **Modelado dimensional** (📕 pendiente): dbt te da las capas pero no te enseña Kimball. El grano, las SCD y el diseño del star schema vienen del Hito 1 y su fuente canónica no está cargada.
- **Estrategia de testing**: la doc lista los tests disponibles pero no prioriza cuáles importan. Esa jerarquía de valor está acá, en este archivo, y viene de experiencia — declarala como tal.
- **Costo**: dbt es agnóstico del warehouse y no habla de créditos. La conexión materialización↔factura está en el Hito 2.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `dbt-project-structure`: auditá tu proyecto real y contame cuántos modelos hay por capa y cuántos violan su capa (marts que leen sources directo, staging con lógica de negocio).
- `dbt-ref-lineage`: grepeá referencias hardcodeadas (`schema.tabla` sin `ref`/`source`) en tu repo. Traeme cuántas hay. Cada una es un nodo perdido.
- `materializations`: elegí el modelo `table` más caro de tu proyecto y calculá cuánto costaría como incremental. Traeme los dos números.
- `incremental-models`: agarrá un incremental tuyo y respondé por escrito las cuatro decisiones (clave, lookback, updates/deletes, frecuencia de reconciliación). Si alguna no tiene respuesta, encontraste un bug.
- `dbt-tests-contracts`: contá cuántos modelos de tu proyecto tienen cero tests y cuántos tienen un test de unicidad sobre su grano. Traeme los dos números.
- `dbt-jinja-macros`: tomá el macro más usado de tu repo, compilá un modelo que lo use y leé el SQL en `target/compiled/`. Traeme qué te sorprendió.

### Para subir a `mastered`

- `incremental-models`: tomá un incremental de producción y probá formalmente que su resultado coincide con el full-refresh sobre la misma ventana. Si no coincide, arreglalo y documentá la causa. Feynman check: explicá por qué filtrar contra `MAX()` del destino pierde datos, sin usar código.
- `dbt-project-structure`: refactorizá un dominio de tu proyecto a las tres capas, con el DAG antes y después. Defendé cada movimiento.
- `dbt-tests-contracts`: definí y aplicá la estrategia de testing de un mart consumido por otro equipo, incluyendo un contract. Feynman check: explicá a un consumidor qué le garantiza el contract y qué NO.
- `materializations`: escribí la guía de decisión de materializaciones de tu equipo (cuándo cada una, con umbrales de costo reales de tu cuenta) y hacela adoptar.

## Anti-patterns que vas a ver en clientes reales

1. **Todo `table`, full-refresh diario**
   - Cómo se hace: es el default más simple y funciona bien los primeros meses.
   - Por qué se hace: nadie mira la factura hasta que alguien la mira.
   - Costo real: horas de compute recreando tablas enormes donde cambia una fracción mínima.
   - Cómo lo arregla un senior: identificar por costo los modelos candidatos, y convertir a incremental los que se pagan — con las cuatro decisiones explícitas, no a las apuradas.

2. **`dbt run` en producción en vez de `dbt build`**
   - Cómo se hace: el job se escribió cuando `run` era lo que se conocía y nunca se revisó.
   - Por qué se hace: inercia, y "los tests los corremos aparte".
   - Costo real: los datos rotos se propagan hasta el dashboard porque nada frena la cadena.
   - Cómo lo arregla un senior: `dbt build` en el job productivo, con severidad calibrada para que los warnings no bloqueen pero los errores sí.

3. **Referencias hardcodeadas**
   - Cómo se hace: `FROM analytics.dim_cliente` porque "así lo veo más claro".
   - Por qué se hace: viene de la costumbre de escribir SQL suelto.
   - Costo real: orden de ejecución impredecible, lineage falso, y modelos de dev que leen y a veces escriben en producción.
   - Cómo lo arregla un senior: `ref()`/`source()` sin excepciones, y una regla de CI que falle si aparece una referencia cruda.

4. **Incremental que filtra contra `MAX()` del destino**
   - Cómo se hace: es el snippet que aparece primero en cualquier búsqueda.
   - Por qué se hace: funciona perfecto mientras los datos lleguen en orden.
   - Costo real: pérdida silenciosa de todo registro que llegue tarde. No hay error, solo faltan filas.
   - Cómo lo arregla un senior: ventana de lookback medida contra la latencia real de la fuente, o estrategia por partición, más un test de reconciliación periódico.

5. **Proyecto sin tests, con documentación autogenerada como coartada**
   - Cómo se hace: se corre `dbt docs` y se muestra el lineage lindo en una reunión.
   - Por qué se hace: la documentación es visible y los tests no.
   - Costo real: el lineage muestra cómo fluye el dato, no si el dato está bien. El primer incidente de calidad lo descubre el negocio.
   - Cómo lo arregla un senior: test de grano en todos los marts, freshness en las sources críticas, y el resto por prioridad de impacto.

6. **Macros por todos lados**
   - Cómo se hace: alguien lee sobre DRY y abstrae todo lo que se repite dos veces.
   - Por qué se hace: buena intención de ingeniería aplicada sin criterio de legibilidad.
   - Costo real: nadie puede leer un modelo sin desenrollar niveles de indirección, y el onboarding de cualquier persona nueva se vuelve un mes.
   - Cómo lo arregla un senior: extraer al tercer uso, nunca antes; y que el macro tenga un nombre que explique la intención de negocio, no la mecánica.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés explicar la responsabilidad única de cada capa y detectar en 5 minutos qué modelos la violan?
- [ ] ¿Podés explicar por qué `ref()` es lo que hace posible el lineage, la separación de ambientes y el CI barato?
- [ ] ¿Podés elegir materialización con un argumento de costo y no de costumbre?
- [ ] ¿Podés enumerar las cuatro decisiones de un incremental y detectar cuál falta en un modelo ajeno?
- [ ] ¿Podés priorizar tests por valor, explicando por qué el de grano vale más que un `not_null`?
- [ ] ¿Podés separar tiempo de compilación de tiempo de ejecución y usar `target/compiled/` como debugger?
- [ ] En entrevista senior, ¿podés contestar "el proyecto dbt tarda 3 horas, ¿qué hacés?" con un diagnóstico ordenado en vez de una lista de tips?
