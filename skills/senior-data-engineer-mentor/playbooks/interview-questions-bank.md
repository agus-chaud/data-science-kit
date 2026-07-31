# Banco de preguntas de entrevista

**Cuándo se carga**: desde `modes/interview.md`, pre-flight check 5.

**Qué contiene**: cinco slots de dificultad creciente por concepto.

| Slot | Nivel | Qué evalúa |
|---|---|---|
| Q1 | mid | Definición + cuándo se usa |
| Q2 | mid-senior | Tradeoff: elegir entre dos opciones con razón |
| Q3 | senior | Anti-pattern o modo de falla |
| Q4 | senior+ | Diagnóstico de un caso real |
| Q5 | staff | Decisión de arquitectura o de costo, defendida |

---

## Cobertura de este banco (declarada, sin letra chica)

**Escrito completo (5 slots)**: 6 conceptos — uno por hito, los de mayor valor de entrevista.
**Generado con protocolo**: los otros 30 conceptos, con las semillas que ya viven en los milestones.

Esto es a propósito. Los milestones ya traen tres preguntas calibradas por concepto (mid / senior / trampa)
con su respuesta esperada: duplicarlas acá sería mantener dos verdades del mismo contenido. El protocolo de
abajo las reutiliza como Q1/Q3/Q5 y genera Q2 y Q4, que son los slots que faltan.

---

# Conceptos con banco completo

## `micro-partitions` (Hito 2)

**Q1 (mid)** — *En Snowflake no hay índices. ¿Cómo evita leer una tabla entera cuando filtrás?*
Espera: bloques inmutables con metadata de rangos por columna; el planificador descarta bloques enteros sin leerlos. Falla si no aparece la idea de descarte.

**Q2 (mid-senior)** — *Tenés una tabla de eventos que se carga cada día ordenada por fecha, y otra que se carga desordenada desde varias fuentes. Ambas se filtran por fecha. ¿Cuál va a rendir mejor y por qué?*
Espera: la cargada ordenada, porque el orden físico de carga determina qué valores caen juntos en cada bloque, y eso es lo que hace efectivo el descarte. La desordenada tiene el rango de fechas repartido en casi todos los bloques.

**Q3 (senior)** — *Un desarrollador filtra con una función aplicada sobre la columna de fecha y la query escanea todo. ¿Qué pasó y cómo lo arreglás?*
Espera: la metadata está sobre la columna cruda, no sobre el resultado de la función, así que el filtro no se puede resolver desde metadata. Fix: reescribir como comparación de rango sobre la columna cruda.

**Q4 (senior+)** — *Una query filtra por una columna con un predicado limpio y aun así escanea el 100% de las particiones. Nadie tocó la tabla. ¿Cómo lo diagnosticás?*
Espera: protocolo ordenado — verificar si el predicado llega al planificador (puede venir de una subquery o un join que no se empuja), revisar si la columna correlaciona con el orden físico (alta cardinalidad cargada desordenada), y recién ahí evaluar clustering. Falla si tira soluciones sin ordenar hipótesis.

**Q5 (staff)** — *Tenés una tabla de 40 TB filtrada por dos columnas distintas según el equipo que consulta. ¿Cómo la diseñás?*
Espera: reconoce que no se puede optimizar el orden físico para dos patrones a la vez, y evalúa opciones con su costo — clustering por el patrón dominante, tablas derivadas por patrón, o vistas materializadas — decidiendo con el volumen de consulta de cada equipo. Falla si dice "clustering key con las dos columnas" sin notar que el orden importa.

## `incremental-models` (Hito 3)

**Q1 (mid)** — *¿Qué hace `is_incremental()` y cuándo devuelve falso?*
Espera: verdadero solo si el modelo es incremental, ya existe físicamente y no se corre con full-refresh. Falso en la primera corrida y en cualquier full-refresh.

**Q2 (mid-senior)** — *¿Cuándo NO convertirías un modelo a incremental aunque sea grande?*
Espera: cuando no hay clave única confiable, cuando los datos se corrigen retroactivamente en profundidad, o cuando la complejidad no se paga contra el costo de reconstrucción. Reconoce que incremental agrega deuda operativa.

**Q3 (senior)** — *¿Por qué filtrar contra el máximo del destino pierde registros?*
Espera: todo registro que llegue con fecha anterior al máximo ya procesado queda fuera para siempre, sin error. Fix: ventana de lookback medida, o estrategia por partición.

**Q4 (senior+)** — *Un mart incremental muestra números distintos a los del sistema fuente. ¿Cómo lo diagnosticás?*
Espera: reconstruir la misma ventana con full-refresh en un esquema paralelo y comparar — eso cuantifica y confirma la divergencia. Después ubicar la causa: filtro sin lookback, clave no única, o transformación no determinística. Falla si empieza a parchear sin medir.

**Q5 (staff)** — *Diseñá la política de incrementales de un equipo de diez personas. ¿Qué reglas ponés?*
Espera: umbral de costo para justificar la conversión, las cuatro decisiones documentadas por modelo, test de reconciliación periódico contra full-refresh, y una cadencia de full-refresh programado. Falla si solo habla de sintaxis.

## `dag-scheduling` (Hito 4)

**Q1 (mid)** — *Un DAG diario corre a las 3 AM. ¿Qué ventana de datos procesa?*
Espera: la ventana anterior; la corrida se dispara al cerrar el intervalo.

**Q2 (mid-senior)** — *¿Cuándo usarías `catchup=True` a propósito?*
Espera: cuando el histórico realmente hay que procesarlo y el pipeline es idempotente y la fuente lo aguanta — siempre con límite de concurrencia. Falla si dice "nunca" sin matiz o "siempre" sin control.

**Q3 (senior)** — *Activaste un DAG y se dispararon cientos de corridas. ¿Qué pasó y cómo lo contenés?*
Espera: catchup con `start_date` viejo. Contención: pausar, limitar concurrencia con pool y `max_active_runs`, decidir si el histórico se procesa, y hacerlo por lotes. Prevención: catchup desactivado por defecto.

**Q4 (senior+)** — *Un pipeline da bien todos los días pero el reproceso de un mes pasado dio números distintos. ¿Qué buscás?*
Espera: dependencias del momento de ejecución en vez de la ventana — fecha del sistema en la transformación, lecturas de tablas que ya cambiaron, o efectos externos. Reconoce que el síntoma "anda bien a diario pero mal al reprocesar" apunta directo a idempotencia.

**Q5 (staff)** — *Definí la política de scheduling de una plataforma con cincuenta DAGs y varios equipos. ¿Qué establecés?*
Espera: catchup desactivado por defecto en la plantilla, límites de concurrencia por DAG y pools por sistema externo, dependencias entre DAGs declaradas por asset y no por horario, y un procedimiento de backfill documentado. Falla si responde por DAG en vez de por plataforma.

## `api-reliability` (Hito 5)

**Q1 (mid)** — *¿Qué es el retroceso exponencial y para qué sirve el jitter?*
Espera: esperar cada vez más entre reintentos; el jitter evita que todos los clientes reintenten sincronizados y vuelvan a tirar el servicio que se recupera.

**Q2 (mid-senior)** — *¿Qué errores reintentás y cuáles no, y por qué?*
Espera: transitorios sí (timeout, 429, 5xx); permanentes no (4xx salvo 429), porque no cambian al repetirlos y solo demoran la detección.

**Q3 (senior)** — *Una escritura da timeout y no sabés si ocurrió. ¿Qué hacés?*
Espera: reintentar con la misma clave de idempotencia; si la API no las soporta, consultar el estado antes de reintentar; si tampoco eso, escalar que la integración no tiene forma segura de reintentar.

**Q4 (senior+)** — *Tu ingesta falla de forma intermitente, siempre después de un rato largo, sin patrón claro de horario. ¿Hipótesis?*
Espera: token obtenido una sola vez al inicio que expira durante la ejecución; o rate limit acumulado; o falta de timeout con degradación de red. Ordena hipótesis por lo que explica el "después de un rato".

**Q5 (staff)** — *Definí el estándar de integración con APIs externas de un equipo. ¿Qué es obligatorio?*
Espera: timeout siempre, política de reintentos diferenciada, respeto del rate limit, claves de idempotencia en escrituras, credenciales resueltas desde almacén, y métricas por integración para poder distinguir "la fuente está caída" de "nuestro código está mal".

## `cicd-for-data` (Hito 6)

**Q1 (mid)** — *¿Por qué no correr todo el proyecto dbt en cada pull request?*
Espera: costo y tiempo desproporcionados; el DAG permite construir solo lo modificado y sus descendientes.

**Q2 (mid-senior)** — *¿Qué diferencia hay entre construir solo lo modificado y construirlo todo, en términos de confianza?*
Espera: si el grafo de dependencias es correcto, lo no modificado no puede cambiar de comportamiento, así que la confianza se mantiene. El riesgo real está en dependencias no declaradas — que es otra razón para no hardcodear referencias.

**Q3 (senior)** — *Tu CI pasa verde desde hace tres meses. ¿Qué revisás?*
Espera: sospecha, no celebración. Datos no representativos, tests que no verifican nada sustancial, o exclusión de los modelos pesados.

**Q4 (senior+)** — *Un cambio pasó el CI y rompió un dashboard en producción. ¿Cómo evitás que se repita?*
Espera: identificar por qué el CI no lo cubría — el consumidor estaba fuera del grafo, el test no existía, o el dato de prueba no tenía el caso. Después cerrar ese hueco específico. Falla si propone "más tests" sin ubicar el hueco.

**Q5 (staff)** — *Diseñá el flujo de promoción de cambios de una plataforma de datos, de dev a producción.*
Espera: esquema por desarrollador, esquema efímero por rama en el CI con construcción selectiva y tests, despliegue al mergear, paso extra con aprobación informada para cambios de alto riesgo, y un dueño de escritura claro por ambiente.

## `dimensional-modeling` (Hito 1)

**Q1 (mid)** — *¿Qué es el grano de una tabla de hechos?*
Espera: qué representa una fila; define qué métricas son sumables y a qué nivel.

**Q2 (mid-senior)** — *¿Cuándo elegís SCD Type 2 en vez de Type 1?*
Espera: cuando el negocio necesita ver el atributo como era al momento del evento. Type 1 si el cambio es una corrección sin valor histórico. Reconoce que Type 2 tiene costo de complejidad.

**Q3 (senior)** — *Un dashboard duplica los montos al agrupar por una dimensión. ¿Qué pasó?*
Espera: granos mezclados en la tabla de hechos, o un join que multiplica filas contra una dimensión Type 2 sin filtrar por vigencia. Diagnostica antes de parchear con `DISTINCT`.

**Q4 (senior+)** — *Heredás un mart sin documentación cuyas métricas no cierran con el sistema fuente. ¿Cómo lo abordás?*
Espera: reconstruir el grano desde los datos (buscar qué combinación de columnas es única), comparar contra la fuente por niveles de agregación para ubicar dónde diverge, y recién ahí decidir si se corrige o se rediseña.

**Q5 (staff)** — *Con warehouses columnares y compute barato, ¿el modelado dimensional sigue teniendo sentido? Defendé tu posición.*
Espera: sí, pero por razones distintas a las originales — ya no es performance, es comprensibilidad para el negocio y consistencia de métricas entre consumidores. Reconoce que la OBT es válida en escenarios de consumidor único. Falla si responde con dogma en cualquiera de las dos direcciones.

---

# Protocolo de generación (para los otros 30 conceptos)

Cuando `modes/interview.md` pide un concepto que no está arriba, generá los 5 slots así:

1. **Abrí el milestone del concepto** y ubicá su bloque "En entrevista te van a preguntar". Trae tres preguntas ya calibradas con su respuesta esperada.
2. **Mapealas directo**:
   - La pregunta *mid* del milestone → **Q1**
   - La pregunta *senior* del milestone → **Q3**
   - La pregunta *trampa* del milestone → **Q5** (son las que exigen defender una posición con matices, que es exactamente el nivel staff)
3. **Generá Q2 (tradeoff)** desde la tabla "Tradeoffs reales" del concepto en el milestone: tomá dos filas contrastantes y preguntá cuándo elige cada una y por qué. Formato: *"¿Cuándo usarías {A} en vez de {B}?"*
4. **Generá Q4 (diagnóstico)** desde la lista de anti-patterns del milestone: tomá uno y presentalo como **síntoma observable**, no como causa. Formato: *"Pasa {síntoma}. ¿Cómo lo diagnosticás?"* La respuesta esperada es un **protocolo ordenado de hipótesis**, no una solución directa — eso es lo que separa senior de mid.
5. **Escribí la respuesta esperada de cada slot** antes de hacer la primera pregunta. Sin criterio escrito de antemano, la evaluación se vuelve impresionista.

## Reglas de calidad de las preguntas generadas

- **Q4 nunca revela la causa en el enunciado.** "La factura subió 40%" es un buen Q4. "El auto-suspend está mal configurado, ¿qué hacés?" no evalúa nada.
- **Q5 tiene que admitir más de una respuesta defendible.** Si hay una sola respuesta correcta, es Q3, no Q5. El nivel staff se evalúa por la calidad de la defensa, no por la elección.
- **Ninguna pregunta se contesta con una definición memorizada** salvo Q1. Si Q2 a Q5 se pueden responder recitando la doc, están mal formuladas.
- **Nada de opción múltiple** salvo que todas las opciones lleven la misma cantidad de palabras (regla de `teach`). Si no podés lograr eso, dejala abierta.
- **Chequeo de versión**: en conceptos de dbt y Airflow, verificá la versión del usuario antes de evaluar. Una respuesta correcta para una versión puede ser incorrecta para otra, y evaluar mal ahí destruye la credibilidad del modo.

## Cuándo promover una pregunta generada al banco

Si una pregunta generada resultó especialmente reveladora — separó bien niveles, o destapó una misconception
que se repite — escribila acá en el formato de arriba y guardala también en engram con
`topic_key: skill/data-engineer-mentor/question-bank/{slug}`. El banco crece por uso, no por planificación.
