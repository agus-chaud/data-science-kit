# Hito 4 — Airflow

> **Chequeo de versión obligatorio antes de enseñar**: Airflow 2.x y 3.x difieren en cosas centrales —
> terminología de scheduling, el Task SDK, versionado de DAGs, assets vs datasets, y comportamiento del
> ejecutor. NUNCA afirmes comportamiento sin saber la versión del usuario. Si no la sabés, preguntá primero
> y verificá contra la doc de ESA versión.

## Por qué importa (perspectiva corporativa)

Airflow es el orquestador de facto de la industria, y también la herramienta que más gente usa con un
modelo mental equivocado. Y el modelo equivocado tiene un nombre: **creer que Airflow es un cron con
interfaz web**.

Si pensás que es un cron, todo lo demás se rompe. No entendés por qué la corrida "de hoy" procesa los datos
de ayer. No entendés por qué al desplegar un DAG nuevo se te dispararon 400 corridas. No entendés por qué
tu backfill produjo datos distintos. No entendés por qué el sensor que espera un archivo te consumió todos
los workers. Todo eso es el mismo malentendido de base.

Y acá está el punto senior: **Airflow no ejecuta tu lógica, la coordina**. Los data engineers que ponen la
transformación pesada dentro del worker de Airflow terminan con un orquestador que es también un motor de
procesamiento, mal dimensionado para las dos cosas. El senior usa Airflow para decidir QUÉ corre, CUÁNDO y
en qué ORDEN, y delega el CÓMO al sistema que corresponde — Snowflake, dbt, un contenedor. Esa distinción
sola te separa del 80%.

## Conceptos de este hito

### airflow-architecture

**Qué es**: Componentes separados. El **scheduler** decide qué tareas están listas y las encola. El
**executor** define cómo/dónde corren (Local, Celery, Kubernetes). Los **workers** ejecutan. La **metadata
database** guarda todo el estado (corridas, tareas, XComs, variables, conexiones). El **triggerer** atiende
las tareas diferidas (deferrable). El **webserver / UI** solo lee y dispara.

**La trampa del junior**: tratar todo como una caja única. Cuando algo falla, no sabe si el problema es del
scheduler (no encoló), del worker (no había slot), de la metadata DB (saturada) o del código del DAG. Y no
sabe que el archivo del DAG se **parsea repetidamente**, así que mete llamadas a APIs o queries en el nivel
superior del archivo y termina castigando la fuente cientos de veces por hora sin que ninguna tarea corra.

**Cómo lo piensa un senior**: hay **dos tiempos que no se mezclan**: el momento en que se PARSEA el archivo
del DAG (frecuente, en el scheduler, tiene que ser rapidísimo y sin efectos) y el momento en que se EJECUTA
la tarea (en un worker, ahí sí hacés trabajo). Todo lo caro va adentro de la función de la tarea, nunca en
el cuerpo del módulo. Y la metadata DB es el cuello de botella real de la mayoría de las instancias grandes:
XComs gordos, retención infinita de corridas y logs en la base la hunden.

**Tradeoffs reales**:

| Executor | Cuándo | Contra |
|---|---|---|
| Local | Dev, instancias chicas | No escala más allá de una máquina |
| Celery | Carga estable, workers de larga vida | Hay que operar broker (Redis/Rabbit) y workers |
| Kubernetes | Cargas heterogéneas, aislamiento por tarea, escala elástica | Overhead de arranque de pod por tarea, más complejidad |
| Servicio gestionado (Astronomer, MWAA, Composer) | Equipo chico, no querés operar Airflow | Costo, menos control, versiones a su ritmo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué hace el scheduler y qué hace el executor?*
  A: El scheduler evalúa el estado de los DAGs y decide qué tareas están listas para correr, marcándolas y encolándolas. El executor define el mecanismo por el cual esas tareas encoladas efectivamente se ejecutan y dónde: en el mismo proceso, en workers Celery, o en pods de Kubernetes.
- Q (senior): *Un DAG está "corriendo" pero ninguna tarea arranca. ¿Qué revisás?*
  A: En orden: si hay slots disponibles (pool agotado, `max_active_tasks` del DAG, `max_active_runs`, parallelism global); si los workers están vivos y consumiendo la cola; si el scheduler está sano y parseando (un archivo de DAG lento puede frenar todo el ciclo); y si la metadata DB está respondiendo. La causa más común en instancias reales es agotamiento de slots por sensores o por un DAG que abrió demasiadas tareas concurrentes.
- Q (trampa): *¿Por qué no conviene poner una llamada a una API en el nivel superior del archivo del DAG?*
  A: Porque ese archivo se parsea una y otra vez para descubrir cambios, independientemente de que el DAG corra. Una llamada en el módulo se ejecuta en cada parseo: castigás la API, hacés lento el parseo del scheduler y, si esa llamada falla, el DAG entero desaparece de la interfaz. Todo lo que tenga efecto o costo va adentro de la tarea.

### dag-scheduling

**Qué es**: Airflow programa sobre **intervalos de datos**, no sobre momentos. Cada corrida tiene un
`data_interval_start` y un `data_interval_end` que definen la ventana de datos que le toca procesar, y
`logical_date` la identifica. La corrida de un intervalo se dispara típicamente **cuando el intervalo
termina**. `catchup` controla si al activar el DAG se generan las corridas pasadas pendientes.

**La trampa del junior**: leerlo como un cron. Cree que el DAG diario de las 03:00 procesa los datos de hoy,
cuando procesa la ventana anterior. Y despliega con `catchup` activo y `start_date` de hace dos años, y le
explota la instancia con cientos de corridas simultáneas contra la fuente productiva.

**Cómo lo piensa un senior**: el modelo mental correcto es **"esta corrida es dueña de esta ventana de
datos"**. De ahí sale todo lo demás: el filtro de la transformación usa los límites de la ventana, no la
fecha de hoy; reprocesar es volver a correr la ventana; y el DAG se vuelve reproducible por diseño. La
segunda regla operativa: `catchup` y `start_date` se deciden juntos y a conciencia. Un `start_date` viejo
con catchup activo no es un detalle, es un incidente programado.

**Tradeoffs reales**:

| Decisión | Efecto | Cuidado |
|---|---|---|
| `catchup=True` | Genera todas las corridas pendientes desde `start_date` | Con fecha vieja, avalancha contra la fuente |
| `catchup=False` | Solo corre desde ahora | Perdés reprocesos automáticos de huecos |
| `max_active_runs=1` | Serializa las corridas del DAG | Necesario si el destino no tolera concurrencia |
| Cron simple | Predecible, conocido por todos | No expresa calendarios de negocio (días hábiles, cierres) |
| Timetable personalizada | Expresa reglas reales de negocio | Código propio a mantener |
| Scheduling por asset (ver `airflow-assets`) | Corre cuando el dato está listo, no por reloj | Requiere que los productores declaren sus assets |

**En entrevista te van a preguntar**:
- Q (mid): *Un DAG diario corre a las 03:00. ¿Qué datos procesa?*
  A: La ventana de datos anterior, no la del día en curso. Airflow dispara la corrida al cerrarse el intervalo, así que la ejecución de la madrugada es dueña del día previo. Por eso el filtro de la transformación debe usar los límites del intervalo y no la fecha actual.
- Q (senior): *Activaste un DAG y se dispararon cientos de corridas. ¿Qué pasó y cómo lo contenés?*
  A: `catchup` activo con un `start_date` muy anterior: Airflow generó todas las corridas pendientes del histórico. Para contenerlo: pauso el DAG, limito la concurrencia (`max_active_runs` y un pool dedicado), y decido si ese histórico realmente hay que procesarlo. Si hay que hacerlo, lo hago controlado por lotes en vez de dejar que el scheduler lo largue todo junto. Para que no vuelva a pasar: `catchup=False` por default y backfills explícitos y acotados.
- Q (trampa): *¿Si uso `datetime.now()` dentro de la tarea, cambia algo mientras el DAG corra a horario?*
  A: Cambia todo en el momento en que necesitás reprocesar. Mientras corre puntual, "hoy" coincide con la ventana y parece funcionar. Cuando hacés un backfill de hace tres meses, `datetime.now()` sigue devolviendo hoy y la tarea procesa la ventana equivocada — produciendo datos incorrectos sin ningún error. Es un bug latente, no un detalle de estilo.

### operators-hooks-taskflow

**Qué es**: **Operators** son plantillas de tarea (ejecutar SQL, lanzar un contenedor, llamar una API);
**hooks** son las interfaces de conexión reutilizables a sistemas externos; **providers** son los paquetes
que traen ambos por tecnología. La **TaskFlow API** (`@task`) permite escribir tareas como funciones Python
con paso de datos implícito vía **XCom**.

**La trampa del junior**: pasar DataFrames por XCom. XCom guarda en la metadata database, así que mover
volumen por ahí hincha la base, la vuelve lenta para todo el mundo y eventualmente falla. La otra trampa:
usar un operator genérico de Python para todo, reimplementando a mano lo que el provider ya resuelve
(reintentos, manejo de credenciales, paginación).

**Cómo lo piensa un senior**: **por XCom viajan referencias, no datos**. Un identificador, una ruta, un
nombre de tabla — nunca el contenido. Los datos van por el storage o el warehouse, que es donde tienen que
estar. Y sobre operators: usar el del provider siempre que exista, porque encapsula el manejo de conexión y
los casos borde que vos vas a descubrir en producción. La TaskFlow API mejora la legibilidad enormemente,
pero no cambia la física de XCom: sigue siendo la metadata DB abajo.

**Tradeoffs reales**:

| Patrón | Cuándo | Contra |
|---|---|---|
| Operator del provider | Default si existe para tu sistema | Dependencia de versión del provider |
| `@task` (TaskFlow) | Lógica Python propia, legibilidad | XCom implícito invita a pasar datos gordos |
| Operator de contenedor (Docker/K8s) | Dependencias propias, aislamiento, otro lenguaje | Overhead de arranque, build de imagen |
| XCom con referencia (id, path) | ✅ Correcto siempre | — |
| XCom con datos | ❌ Nunca en volumen | Hincha la metadata DB, límite de tamaño |
| Custom XCom backend (a object storage) | Necesitás pasar payloads medianos | Complejidad extra, otro punto de falla |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es XCom y cuál es su límite?*
  A: El mecanismo por el que una tarea le pasa un valor a otra. Se guarda en la metadata database, así que está pensado para valores chicos: identificadores, rutas, banderas. Mover datasets por ahí degrada la base de metadata de toda la instancia.
- Q (senior): *Necesitás pasar el resultado de un query de un millón de filas de una tarea a otra. ¿Cómo?*
  A: No lo paso. La primera tarea materializa el resultado donde corresponde — una tabla del warehouse o un archivo en object storage — y por XCom viaja solo la referencia: el nombre de la tabla o la ruta. La segunda tarea lee desde ahí. El orquestador coordina, no transporta datos.
- Q (trampa): *¿Conviene escribir un operator propio para llamar a una API interna?*
  A: Solo si el patrón se repite en varios DAGs y encapsula algo real (autenticación, paginación, reintentos específicos). Para un uso único, un `@task` con un hook existente es más simple y más legible. Un operator propio es código que hay que versionar, testear y mantener alineado con la versión de Airflow — y la mayoría de los operators propios que uno ve en la calle son un `requests.get` disfrazado.

### dag-idempotency

**Qué es**: Que reejecutar una tarea sobre la misma ventana produzca el mismo resultado. En Airflow se
apoya en usar las **variables templadas del intervalo** (`{{ data_interval_start }}`,
`{{ data_interval_end }}`, `{{ ds }}`) en vez de fechas del sistema, y en escrituras que toleren repetición.

**La trampa del junior**: `CURRENT_DATE`, `datetime.now()` o `NOW()` adentro de la tarea o del SQL. Anda
perfecto hasta el primer reintento o el primer backfill, y ahí produce datos silenciosamente equivocados.

**Cómo lo piensa un senior**: **un reintento tiene que ser gratis**. Si reintentar una tarea da miedo, la
tarea está mal escrita, porque Airflow reintenta por diseño — ante un timeout de red, un worker que se
muere, un 429 de la fuente. Y de ahí sale la regla que ordena el hito entero: la fecha viene del intervalo,
la escritura es merge o reemplazo de partición, y los efectos externos (mandar un mail, disparar un
webhook) se aíslan en tareas propias al final, porque esos SÍ son irrepetibles.

**Tradeoffs reales**:

| Patrón | Idempotente | Nota |
|---|---|---|
| `{{ data_interval_start }}` en el filtro | ✅ | El default correcto |
| `datetime.now()` / `CURRENT_DATE` | ❌ | Rompe todo backfill y todo reintento |
| Escritura con merge por clave | ✅ | Requiere clave real y única |
| Reemplazo de partición completa | ✅ | La partición debe alinearse con la ventana |
| `INSERT` directo | ❌ | Duplica en el primer reintento |
| Efecto externo (mail, webhook, pago) | ❌ inherente | Aislar en tarea final, con control de repetición propio |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué usar `{{ ds }}` en vez de `CURRENT_DATE`?*
  A: Porque `{{ ds }}` viene del intervalo de la corrida, así que un reintento o un backfill procesan exactamente la misma ventana que la corrida original. `CURRENT_DATE` devuelve el día de ejecución real, con lo cual un reproceso del pasado procesaría la ventana equivocada sin dar error.
- Q (senior): *¿Cómo hacés idempotente una tarea que manda un email de reporte?*
  A: Estrictamente no puedo hacerla idempotente, porque el efecto es externo e irreversible. Lo que hago es contenerla: la aíslo como última tarea del DAG, separada de las que producen datos, y le pongo un control propio — una marca de "ya enviado para esta ventana" en una tabla, chequeada antes de enviar. Así el reintento del DAG no reenvía. La regla general es que los efectos irreversibles no se mezclan con las tareas de transformación.
- Q (trampa): *Tu tarea usa `{{ ds }}` correctamente. ¿Ya es idempotente?*
  A: Todavía no necesariamente. La fecha correcta resuelve QUÉ ventana se procesa, pero si la escritura es un `INSERT` sin clave, el reintento duplica igual. Y si la transformación usa algo no determinístico o depende de una tabla que cambió entre corridas, el resultado varía. Idempotencia es de la tarea completa: entrada, transformación y escritura.

### airflow-assets

**Qué es**: Programar por **datos disponibles** en lugar de por reloj. Un DAG declara que produce un asset;
otro declara que depende de ese asset y se dispara cuando el productor lo actualiza. (La terminología y el
alcance cambiaron entre versiones: *datasets* en Airflow 2.4+, *assets* con más capacidades en Airflow 3 —
verificá la versión antes de enseñar detalles.)

**La trampa del junior**: encadenar DAGs por horario. "El de ingesta corre a las 2, así que el de
transformación lo pongo a las 2:30 que ya va a estar". Un día la ingesta tarda 40 minutos y el downstream
procesa datos incompletos. Sin error. Sin alerta. Solo números mal.

**Cómo lo piensa un senior**: **el reloj es una suposición; el asset es un hecho**. Encadenar por tiempo
funciona hasta la primera vez que algo tarda más, y falla de la peor manera posible: silenciosamente y con
datos parciales. El scheduling por asset convierte esa suposición implícita en una dependencia declarada, y
como efecto secundario documenta el lineage a nivel de orquestación. La alternativa clásica (un sensor que
espera al otro DAG) funciona pero consume recursos esperando y acopla los DAGs de forma más frágil.

**Tradeoffs reales**:

| Patrón de encadenamiento | Confiabilidad | Costo | Nota |
|---|---|---|---|
| Horario "con margen" | ❌ Baja | Nulo | Falla silenciosa cuando el upstream se atrasa |
| Sensor externo esperando al otro DAG | Media | Ocupa slot (salvo deferrable) | Acopla, y el sensor puede quedar colgado |
| Trigger explícito del DAG downstream | Alta | Bajo | Acopla el productor al consumidor por nombre |
| Scheduling por asset | Alta | Bajo | Desacopla: el productor declara, el consumidor se suscribe |
| Un solo DAG con todas las tareas | Alta | Bajo | Simple, pero el DAG crece y mezcla ownership de equipos |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué problema resuelve el scheduling por assets?*
  A: El de encadenar DAGs por horario. En vez de suponer que el upstream ya terminó, el consumidor se dispara cuando el productor efectivamente actualizó el dato. Elimina la clase de fallas donde el downstream corre con datos incompletos sin que nada falle.
- Q (senior): *¿Cuándo preferirías un solo DAG grande en vez de dos DAGs encadenados por asset?*
  A: Cuando ambas partes tienen el mismo dueño, el mismo SLA y la misma cadencia. Dividir en dos DAGs tiene sentido cuando hay equipos distintos, frecuencias distintas, o varios consumidores del mismo producto. La división por asset es una decisión organizacional además de técnica: define fronteras de responsabilidad.
- Q (trampa): *Con assets, ¿ya no necesitás sensores?*
  A: Para dependencias entre DAGs de tu propia instancia, en gran medida sí los reemplaza. Pero seguís necesitando esperar cosas que Airflow no produce: un archivo de un tercero, un job en otro sistema, la disponibilidad de una API. Ahí el sensor sigue siendo la herramienta — preferentemente en modo diferido para no ocupar un worker mientras espera.

### airflow-scaling

**Qué es**: Los límites de concurrencia y cómo se reparten. **Pools** limitan tareas por recurso compartido;
`max_active_tasks` acota un DAG; `max_active_runs` acota corridas simultáneas del mismo DAG; hay un límite
global de paralelismo. Los **deferrable operators** liberan el worker mientras esperan, delegando la espera
al triggerer.

**La trampa del junior**: llenar la instancia de sensores clásicos que ocupan un worker completo cada uno
mientras no hacen nada más que preguntar. Con veinte sensores esperando archivos, la instancia se queda sin
slots y las tareas que sí trabajan hacen cola. Se ve exactamente como "Airflow está lento" y no lo está: está
lleno de tareas dormidas.

**Cómo lo piensa un senior**: **los slots son el recurso escaso y hay que protegerlos**. Toda tarea que
espera algo debe ser diferida o, como mínimo, estar en modo que libere el worker entre chequeos. Los pools
existen para proteger a los sistemas de AFUERA, no a Airflow: si tu fuente aguanta cinco conexiones
concurrentes, el pool es el lugar donde se declara ese límite, y así el orquestador no la tira abajo por más
DAGs que agregues. Y la regla que resume el hito: si tu worker de Airflow está usando CPU, probablemente
estás haciendo el trabajo en el lugar equivocado.

**Tradeoffs reales**:

| Palanca | Protege | Cuándo tocarla |
|---|---|---|
| `pool` | Sistemas externos (fuente, warehouse) | Siempre que varios DAGs golpeen el mismo recurso |
| `max_active_runs` | El destino, ante corridas simultáneas | Backfills, DAGs que escriben la misma tabla |
| `max_active_tasks` | La instancia, ante un DAG muy ancho | DAGs con cientos de tareas paralelas |
| Deferrable operators | Slots de workers | Toda espera de más de unos minutos |
| Sensor en modo que libera el worker | Slots | Si no hay versión deferrable disponible |
| Más workers | Throughput real | Solo después de descartar tareas dormidas |

**En entrevista te van a preguntar**:
- Q (mid): *¿Para qué sirve un pool?*
  A: Para limitar cuántas tareas pueden usar simultáneamente un recurso compartido. Se usa sobre todo para proteger sistemas externos: si la base fuente tolera cinco conexiones, se declara un pool de cinco y todas las tareas que la tocan lo comparten, sin importar de qué DAG vengan.
- Q (senior): *"Airflow está lento" y agregar workers no mejoró nada. ¿Qué pasa?*
  A: Si sumar capacidad no cambia nada, el cuello no es de cómputo. Miro cuántas tareas están ocupando slots sin hacer trabajo: sensores clásicos esperando, tareas colgadas sin timeout. También reviso si el problema es el scheduler (parseo lento por archivos de DAG pesados) o la metadata DB saturada. Agregar workers a una instancia llena de tareas dormidas es pagar por más asientos en una sala de espera.
- Q (trampa): *¿Un deferrable operator hace que la tarea corra más rápido?*
  A: No, tarda lo mismo. Lo que cambia es que mientras espera no ocupa un slot de worker: el estado se delega al triggerer, que atiende muchas esperas concurrentes de forma liviana. El beneficio es de capacidad de la instancia completa, no de latencia de esa tarea.

## Lo que la doc oficial cubre bien acá

- **Core Concepts** (https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html) — arquitectura y componentes.
- **Authoring and Scheduling** (https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html) — intervalos de datos, catchup, timetables, assets. Es la sección que resuelve el malentendido central.
- **Templates Reference** (https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html) — todas las variables templadas disponibles. La página más práctica de toda la doc.
- **Deferring** (https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html) — cómo funciona el triggerer.
- **Best Practices** (https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html) — top-level code, idempotencia, testing de DAGs.
- **Astronomer Learn** (https://docs.astronomer.io/learn) — el mejor material pedagógico de Airflow que existe, escrito por gente que mantiene el proyecto. Cuando la doc oficial es árida, esto es lo que se lee.
- **Astronomer Registry** (https://registry.astronomer.io/) — buscador de providers y operators, mejor que rastrear GitHub.

## Gaps

- **Diseño de pipelines a nivel sistema** (📕 pendiente): cuándo dividir DAGs, cómo repartir ownership entre equipos, SLAs de datos. La doc cubre mecánica, no diseño organizacional.
- **Airflow 3 vs 2**: la doc de cada versión es correcta, pero no hay una guía única que te diga "esto que sabías cambió". Por eso el chequeo de versión es obligatorio en este hito.
- **Costo**: Airflow no habla del costo de lo que orquesta. La conexión frecuencia↔créditos está en el Hito 2.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `airflow-architecture`: averiguá qué executor usa tu instancia, cuántos workers tiene y dónde vive la metadata DB. Traeme los tres datos.
- `dag-scheduling`: agarrá un DAG diario tuyo y decime qué ventana de datos procesa la corrida de esta madrugada. Después verificalo en la interfaz.
- `operators-hooks-taskflow`: buscá en tus DAGs un XCom que pase algo más grande que un identificador. Traeme el caso.
- `dag-idempotency`: grepeá `datetime.now()`, `CURRENT_DATE` y `NOW()` en tus DAGs y en el SQL que disparan. Traeme cuántos hits hay. Cada uno es un backfill roto.
- `airflow-assets`: encontrá dos DAGs tuyos encadenados por horario "con margen". Traeme cuáles son y qué pasa si el primero tarda el doble.
- `airflow-scaling`: contá cuántos sensores no diferidos corren en paralelo en tu instancia en hora pico. Traeme el número y cuántos slots totales hay.

### Para subir a `mastered`

- `dag-idempotency`: tomá un DAG productivo no idempotente y convertilo, incluyendo la prueba de que un reproceso da el mismo resultado. Feynman check: explicá por qué la fecha tiene que venir de afuera de la tarea, sin usar la palabra "template".
- `dag-scheduling`: ejecutá un backfill controlado de una ventana real, con límite de concurrencia y validación por lote. Documentá el procedimiento como runbook del equipo.
- `airflow-assets`: convertí un encadenamiento por horario a dependencia por asset en tu instancia y medí cuántas fallas silenciosas eliminaba ese cambio.
- `airflow-scaling`: diagnosticá un problema real de capacidad de tu instancia y arreglalo sin agregar workers. Feynman check: explicá la diferencia entre "lleno" y "saturado".

## Anti-patterns que vas a ver en clientes reales

1. **DAGs encadenados por horario con margen**
   - Cómo se hace: "el de ingesta a las 2, el de transformación a las 2:30".
   - Por qué se hace: es lo primero que se le ocurre a cualquiera y funciona el 95% de los días.
   - Costo real: el 5% restante procesa datos incompletos sin ningún error. El negocio descubre el problema, no el equipo.
   - Cómo lo arregla un senior: dependencia por asset, o trigger explícito, o unificar en un DAG. Nunca por reloj.

2. **`catchup=True` con `start_date` viejo**
   - Cómo se hace: se copia un DAG existente y se cambia el nombre, sin tocar la fecha ni la bandera.
   - Por qué se hace: la interacción entre esos dos parámetros no es obvia hasta que te muerde.
   - Costo real: cientos de corridas simultáneas contra la fuente productiva. Incidente de verdad, no molestia.
   - Cómo lo arregla un senior: `catchup=False` por default en la plantilla del equipo, backfills explícitos y acotados, `max_active_runs` como red de seguridad.

3. **Transformación pesada dentro del worker**
   - Cómo se hace: se traen los datos a pandas en el worker, se transforman y se escriben de vuelta.
   - Por qué se hace: es cómodo y en dev con mil filas anda perfecto.
   - Costo real: el worker se queda sin memoria con volumen real, el orquestador se convierte en motor de procesamiento mal dimensionado, y no escala.
   - Cómo lo arregla un senior: empujar el cómputo al warehouse (SQL, dbt) y dejar a Airflow coordinando. Si hace falta Python sobre volumen, va en un contenedor dedicado, no en el worker.

4. **Sensores clásicos por todos lados**
   - Cómo se hace: cada espera de archivo o de tabla se resuelve con un sensor que pregunta cada minuto.
   - Por qué se hace: es el operator que aparece primero y funciona.
   - Costo real: la instancia se llena de tareas dormidas ocupando slots. Todo parece lento y agregar workers no arregla nada.
   - Cómo lo arregla un senior: deferrable, o al menos modo que libere el worker, más un timeout siempre.

5. **XCom como transporte de datos**
   - Cómo se hace: `return df` desde una tarea TaskFlow.
   - Por qué se hace: la TaskFlow API hace que pasar valores sea tan natural que no se piensa dónde se guardan.
   - Costo real: la metadata DB crece, se pone lenta para toda la instancia, y en algún momento la tarea falla por tamaño.
   - Cómo lo arregla un senior: materializar en warehouse u object storage y pasar la referencia.

6. **Código caro en el nivel superior del archivo del DAG**
   - Cómo se hace: una query o una llamada a API afuera de cualquier tarea, para "armar la lista de tablas".
   - Por qué se hace: parece la forma natural de generar tareas dinámicamente.
   - Costo real: se ejecuta en cada parseo, castiga la fuente, ralentiza el scheduler y si falla desaparece el DAG entero.
   - Cómo lo arregla un senior: mover la generación dinámica al momento de ejecución (mapeo dinámico de tareas) o leer la lista desde una variable/archivo barato de parsear.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés separar tiempo de parseo de tiempo de ejecución y decir qué va en cada uno?
- [ ] ¿Podés explicar qué ventana de datos procesa una corrida cualquiera, sin dudar?
- [ ] ¿Podés detectar en un DAG ajeno, en 5 minutos, qué lo hace no idempotente?
- [ ] ¿Podés explicar por qué encadenar por horario es una falla silenciosa esperando?
- [ ] ¿Podés diagnosticar "Airflow está lento" distinguiendo cuello de slots, de scheduler y de metadata DB?
- [ ] ¿Podés decir qué NO debería correr dentro de un worker de Airflow y por qué?
- [ ] En entrevista senior, ¿podés contestar "hay que backfillear 6 meses" con un procedimiento controlado en vez de "le pongo catchup"?
