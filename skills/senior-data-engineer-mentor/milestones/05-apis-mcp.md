# Hito 5 — APIs & MCP

## Por qué importa (perspectiva corporativa)

Acá está el hito que casi ningún data engineer estudia y que casi todos necesitan. Porque tu trabajo real es
así: la mitad de tus fuentes son APIs, y la mitad de tus consumidores quieren consumir por API. Estás
parado en el medio de dos contratos que no diseñaste y que no terminás de entender — y cuando algo se rompe
en esa frontera, la conversación con el equipo de backend la ganás o la perdés según si hablás su idioma.

Y hay una cosa que no se dice lo suficiente: **el data engineer que entiende diseño de APIs deja de ser el
que pide y pasa a ser el que negocia**. Cuando podés decir "este endpoint no me sirve porque pagina por
offset y la tabla tiene inserts concurrentes, necesito cursor" — con esa frase cambiaste de categoría en la
reunión. Y cuando podés diseñar vos el contrato de salida de tus datos, dejás de ser un proveedor de tablas
y pasás a ser dueño de un producto de datos.

**MCP** es la pata nueva y es la que más rápido va a valer. Model Context Protocol es el estándar para
conectar modelos de lenguaje a herramientas y fuentes de datos. Para un data engineer eso significa algo muy
concreto: exponer tu warehouse, tu catálogo o tus métricas a un agente de forma controlada y auditada. El que
sepa hacer eso bien en 2026 va a estar armando la capa que todas las empresas van a necesitar. Y es la
misma disciplina de siempre — contratos, permisos, límites — aplicada a un consumidor nuevo.

## Conceptos de este hito

### rest-resource-design

**Qué es**: Modelar la API alrededor de **recursos** (sustantivos: `/pedidos`, `/clientes/{id}/facturas`)
con un conjunto acotado de **métodos estándar** (listar, obtener, crear, actualizar, borrar) y códigos de
estado con semántica real.

**La trampa del junior**: diseñar la API como una lista de funciones RPC disfrazadas de URL:
`/obtenerPedidosDelCliente`, `/actualizarEstadoPedido`. Todo por POST, todo devolviendo 200 con un campo
`error` adentro del cuerpo. Funciona, y a la vez rompe todo lo que la infraestructura HTTP te daba gratis:
caché, reintentos, monitoreo por código de estado.

**Cómo lo piensa un senior**: la disciplina de recursos no es estética, es **interoperabilidad**. Un GET
declara que es seguro y cacheable; un PUT declara que es idempotente; un 404 y un 500 significan cosas
distintas para el balanceador, para el cliente y para el reintentador automático. Cuando metés todo en POST
con 200, tu API deja de participar del ecosistema y cada cliente tiene que aprender tus reglas propias. Para
un consumidor de datos eso se traduce en pipelines más frágiles y más código de plomería.

**Tradeoffs reales**:

| Decisión | Opción | Cuándo |
|---|---|---|
| Estilo | REST orientado a recursos | Default para APIs públicas y de integración |
| Estilo | RPC (gRPC) | Servicio a servicio interno, alto rendimiento, contrato fuerte |
| Estilo | GraphQL | Muchos clientes con necesidades de campos muy distintas |
| Acción que no es CRUD | Sub-recurso o método custom explícito | Cancelar, aprobar, recalcular |
| Errores | Código de estado + cuerpo estructurado | Siempre |
| Errores | 200 con `error` adentro | ❌ Nunca — rompe reintentos y monitoreo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué importa usar el verbo HTTP correcto?*
  A: Porque el verbo comunica garantías que toda la infraestructura usa: GET es seguro y cacheable, PUT y DELETE son idempotentes, POST no lo es. Los proxies, los reintentadores y los clientes toman decisiones a partir de eso. Si mandás todo por POST, nadie puede reintentar con seguridad ni cachear nada.
- Q (senior): *¿Cómo modelás una acción que no es CRUD, como "cancelar un pedido"?*
  A: Dos opciones defendibles. Como transición de estado sobre el recurso, actualizando su campo de estado — sirve si el modelo lo permite y hay un solo camino. O como sub-recurso explícito de la acción, tipo `POST /pedidos/{id}/cancelaciones`, que además te deja registrar quién y cuándo canceló. Lo que no hago es inventar un verbo en la URL, porque rompe la uniformidad que hace predecible al resto de la API.
- Q (trampa): *¿Un 200 con un campo `success: false` es aceptable si el cliente lo entiende?*
  A: Lo entiende ese cliente, pero nadie más. El balanceador ve un éxito, el monitoreo no cuenta el error, el reintentador automático no reintenta, y el dashboard de disponibilidad miente. Los códigos de estado son la interfaz con toda la infraestructura intermedia, no solo con el desarrollador que escribe el cliente.

### api-contracts

**Qué es**: **OpenAPI** es la especificación que describe formalmente la API: rutas, parámetros, esquemas de
request y response, errores, autenticación. **Contract-first** significa acordar la spec antes de escribir
código de ambos lados.

**La trampa del junior**: descubrir la API leyendo respuestas reales y programando contra lo que ve. Sin
spec, cualquier cambio del proveedor es una sorpresa y la única forma de enterarse es que el pipeline se
rompa en producción.

**Cómo lo piensa un senior**: el contrato es **el punto de sincronización entre equipos que no se
coordinan a diario**. Con OpenAPI acordado, back y front (o back y data) trabajan en paralelo desde el día
uno, se generan clientes y mocks automáticamente, y el CI puede detectar cambios que rompen antes del
deploy. Y la distinción que hay que tener clarísima: **agregar un campo opcional no rompe; sacar un campo,
cambiar un tipo o volver obligatorio algo que no lo era, sí rompe**. Esa línea es la que define si necesitás
versionar o no.

**Tradeoffs reales**:

| Estrategia de versionado | Cuándo | Contra |
|---|---|---|
| En la ruta (`/v1/`, `/v2/`) | Default, visible y simple | Duplicación de rutas, migración explícita |
| Por header | URLs estables | Menos visible, más fácil de olvidar |
| Sin versión, solo cambios compatibles | APIs internas con consumidores conocidos | Requiere disciplina real y catálogo de consumidores |
| Campos deprecados con aviso y plazo | Complemento de cualquier estrategia | Requiere saber quién consume qué |

| Cambio | ¿Rompe? |
|---|---|
| Agregar campo opcional en la respuesta | ❌ No (si el cliente ignora lo desconocido) |
| Agregar parámetro opcional | ❌ No |
| Sacar un campo de la respuesta | ✅ Sí |
| Cambiar el tipo de un campo | ✅ Sí |
| Hacer obligatorio un parámetro que era opcional | ✅ Sí |
| Cambiar el significado de un valor sin cambiar el tipo | ✅ Sí, y es el peor porque es invisible |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es contract-first y qué gana el equipo?*
  A: Acordar la especificación de la API antes de implementarla. Gana paralelismo: el consumidor genera un cliente y un mock desde la spec y avanza sin esperar al proveedor. Y gana una fuente de verdad única para discutir cambios antes de que existan en código.
- Q (senior): *¿Cómo evitás romper consumidores al evolucionar una API?*
  A: Primero, sé quiénes son mis consumidores — sin ese catálogo cualquier política es teórica. Después, regla de solo-cambios-compatibles por default: agregar es libre, sacar o cambiar tipos requiere versión nueva. Marco deprecaciones con plazo explícito y las mido: si nadie usa el campo viejo, se saca. Y pongo una validación en el CI que compara la spec contra la anterior y falla si el diff es incompatible, para que la regla no dependa de que alguien se acuerde.
- Q (trampa): *Agregaste un campo nuevo en la respuesta. ¿Seguro que no rompe a nadie?*
  A: No es seguro. Rompe a los clientes con parseo estricto que fallan ante campos desconocidos, y a los que hacen validación de esquema cerrada. Es el caso más común de "cambio compatible" que igual rompe. Por eso la regla completa incluye que los consumidores toleren campos desconocidos — y eso es parte del contrato, no una suposición.

### api-pagination-filtering

**Qué es**: Cómo devolver conjuntos grandes en pedazos. **Offset** salta N filas y devuelve las siguientes M.
**Cursor** devuelve un puntero opaco a la posición y el cliente pide "lo que sigue después de esto".

**La trampa del junior**: paginar por offset sobre una tabla que recibe inserts mientras vos leés. Entre la
página 1 y la 2 entraron filas nuevas, todo se corrió, y terminás con registros duplicados o salteados. La
ingesta no falla, solo trae mal los datos — y descubrirlo lleva meses.

**Cómo lo piensa un senior**: **offset es correcto solo sobre datos congelados**. En cuanto hay escritura
concurrente, la única paginación consistente es por cursor sobre un orden estable, y el orden tiene que ser
único (una fecha sola no alcanza si hay empates: se desempata con el identificador). Y del lado de quien
consume una API ajena, la primera pregunta al leer la doc es: *¿qué garantía de consistencia me da la
paginación?*. Si la doc no lo dice, hay que asumir lo peor y diseñar la ingesta a la defensiva —
deduplicando por clave al escribir.

**Tradeoffs reales**:

| Estrategia | Consistente con escrituras | Permite saltar a página N | Cuándo |
|---|---|---|---|
| Offset / page number | ❌ No | ✅ Sí | UI con paginador numerado sobre datos estables |
| Cursor / keyset | ✅ Sí | ❌ No | Ingesta de datos, feeds, cualquier volumen |
| Snapshot / punto en el tiempo | ✅ Sí | Depende | Exportaciones grandes, si el proveedor lo soporta |
| Sin paginación | — | — | Solo con máximo acotado y garantizado |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué cursor es mejor que offset para ingesta?*
  A: Porque el offset se calcula sobre el conjunto en el momento de cada consulta. Si entran o salen filas entre páginas, las posiciones se desplazan y terminás duplicando o salteando registros. El cursor apunta a una posición estable del orden, así que "lo que sigue" es lo que sigue de verdad.
- Q (senior): *Consumís una API que solo pagina por offset y no podés cambiarla. ¿Cómo protegés la ingesta?*
  A: Asumo que va a duplicar y a saltear, y compenso: escribo con merge por clave única para que los duplicados sean inocuos, y agrego una ventana de solapamiento en cada corrida para reducir el riesgo de salteo. Después valido con una reconciliación de conteos contra la fuente, porque el salteo no se detecta solo. Y dejo documentado que es una limitación de la fuente, para que sea un ticket contra el proveedor y no una deuda invisible mía.
- Q (trampa): *Ordenás por `fecha_creacion` y paginás por cursor. ¿Es consistente?*
  A: No necesariamente. Si hay varios registros con la misma fecha, el orden entre ellos no está definido y puede variar entre consultas, con lo cual el cursor puede repetir o saltear en los empates. El orden tiene que ser total: fecha más un desempate único como el identificador.

### api-auth

**Qué es**: Cómo el pipeline prueba quién es. Para máquina a máquina, el estándar es **OAuth 2.0 client
credentials**: la aplicación intercambia un identificador y un secreto por un token de vida corta con
**scopes** acotados. Alternativas: API keys (simples, sin expiración), identidades gestionadas de la nube
(sin secreto que rotar), certificados.

**La trampa del junior**: la API key en el código, en el DAG, o en un archivo `.env` que terminó
commiteado. Y sin rotación, porque rotarla implicaría saber quién la usa, y nadie sabe.

**Cómo lo piensa un senior**: **el secreto no se guarda, se resuelve en tiempo de ejecución**. Vive en un
almacén de secretos (Key Vault) y el proceso lo obtiene con su propia identidad. Mejor todavía: identidad
gestionada o federación de credenciales, donde directamente no hay secreto que rotar. Y sobre permisos, la
regla es de mínimo privilegio con alcance por ambiente: la credencial de producción no funciona en dev, y la
del pipeline de ingesta no puede escribir en marts. Cuando eso no se cumple, el radio de explosión de
cualquier incidente es toda la plataforma.

**Tradeoffs reales**:

| Mecanismo | Seguridad | Operación | Cuándo |
|---|---|---|---|
| API key en variable de entorno | Baja | Mínima | Solo desarrollo, o APIs sin datos sensibles |
| API key en almacén de secretos | Media | Baja | Cuando el proveedor no ofrece otra cosa |
| OAuth 2.0 client credentials | Alta | Media (rotación de secreto) | Estándar para máquina a máquina |
| Identidad gestionada / federación | Alta | Mínima (no hay secreto) | Default cuando el cloud lo permite |
| Certificado cliente (mTLS) | Muy alta | Alta | Integraciones con requisitos regulatorios |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es el flujo client credentials y cuándo se usa?*
  A: Un flujo de OAuth 2.0 donde la aplicación se autentica con su propio identificador y secreto contra el servidor de autorización y recibe un token de acceso de vida corta. Se usa para comunicación máquina a máquina, donde no hay un usuario final que dé consentimiento.
- Q (senior): *¿Cómo diseñás la gestión de credenciales de una plataforma de datos?*
  A: Identidad por carga de trabajo, no una credencial compartida — así puedo revocar y auditar por proceso. Los secretos viven en un almacén central y se resuelven en ejecución, nunca en el repo ni en la definición del pipeline. Donde el cloud lo permita, identidad gestionada para eliminar el secreto. Permisos de mínimo privilegio separados por ambiente, y rotación automatizada, porque una rotación que depende de que alguien se acuerde no ocurre.
- Q (trampa): *El token expira cada hora y tu job dura tres. ¿Qué hacés?*
  A: Renovarlo durante la ejecución, no pedirlo una sola vez al inicio. El error clásico es obtener el token al arrancar y guardarlo en una variable: el job falla a la hora exacta, de forma intermitente y difícil de reproducir en pruebas cortas. El cliente HTTP tiene que refrescar el token cuando está por vencer o ante un 401, con reintento.

### api-reliability

**Qué es**: Lo que hace que una integración sobreviva a la realidad: **timeouts** siempre, **reintentos** con
retroceso exponencial y jitter, respeto de **rate limits** (429 y su indicación de espera), **claves de
idempotencia** para que reintentar una escritura no duplique, y errores en un formato estructurado estándar
(RFC 9457, *Problem Details*).

**La trampa del junior**: un `requests.get` sin timeout y sin reintentos. El día que la red se pone lenta,
la tarea queda colgada indefinidamente ocupando un worker, y nadie se entera hasta que alguien mira por qué
el DAG lleva 14 horas.

**Cómo lo piensa un senior**: **toda llamada de red falla, la pregunta es qué hacés cuando falla**. Y la
respuesta se divide en dos: los errores transitorios (timeout, 429, 5xx) se reintentan con retroceso y
jitter — el jitter importa porque sin él todos los clientes reintentan sincronizados y golpean juntos al
servicio que se está recuperando. Los errores permanentes (400, 401, 404) NO se reintentan: reintentar un
400 es quemar cuota para recibir el mismo 400. Y para escrituras, la clave de idempotencia es lo que hace
que reintentar sea seguro — sin ella, un timeout te deja sin saber si la operación ocurrió, y reintentar
puede duplicar.

**Tradeoffs reales**:

| Situación | Acción correcta | Error común |
|---|---|---|
| Timeout de conexión | Reintentar con retroceso | No poner timeout, colgarse para siempre |
| 429 (rate limit) | Esperar lo que indica el servidor y reintentar | Reintentar inmediato y empeorar el bloqueo |
| 5xx | Reintentar con retroceso + jitter | Reintentar en bucle cerrado |
| 4xx (salvo 429) | No reintentar, fallar y alertar | Reintentar y quemar cuota |
| Escritura que dio timeout | Reintentar con la misma clave de idempotencia | Reintentar sin clave y duplicar |
| Muchos clientes reintentando | Jitter obligatorio | Retroceso sin jitter → estampida sincronizada |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es el retroceso exponencial y por qué se le agrega jitter?*
  A: Esperar cada vez más entre reintentos para no golpear un servicio caído. El jitter es una variación aleatoria sobre esa espera: sin él, todos los clientes que fallaron al mismo tiempo reintentan al mismo tiempo, y el servicio que se estaba recuperando recibe una avalancha sincronizada que lo vuelve a tirar.
- Q (senior): *Tu tarea escribe en una API, da timeout, y no sabés si la escritura ocurrió. ¿Qué hacés?*
  A: Si la API soporta claves de idempotencia, reintento con la misma clave: el servidor reconoce que es la misma operación y no la duplica. Si no las soporta, tengo que consultar el estado antes de reintentar — y si tampoco hay forma de consultar, el diseño de esa integración tiene un problema que hay que escalar al proveedor, porque no existe forma segura de reintentar.
- Q (trampa): *¿Conviene reintentar todos los errores para que el pipeline sea más resiliente?*
  A: No. Reintentar un error permanente no lo arregla: un 401 va a seguir siendo 401 y un 400 también. Lo único que lográs es demorar la detección del problema real, consumir cuota y a veces disparar bloqueos por abuso. Los reintentos son para lo transitorio; lo permanente tiene que fallar rápido y ruidoso.

### mcp-protocol

**Qué es**: **Model Context Protocol**: estándar abierto para que un cliente con modelo de lenguaje se
conecte a servidores que exponen capacidades. Tres primitivas: **tools** (funciones ejecutables con efectos),
**resources** (datos leíbles identificados por URI) y **prompts** (plantillas que el servidor ofrece).
Cliente y servidor negocian capacidades al conectarse, sobre distintos transportes.

**La trampa del junior**: confundir MCP con "llamar funciones desde un LLM". Eso es una parte. MCP es el
protocolo de **descubrimiento y ejecución** entre aplicaciones y servidores: un servidor escrito una vez lo
usa cualquier cliente compatible, sin reescribir la integración por cada proveedor de modelo.

**Cómo lo piensa un senior**: para un data engineer, MCP es **la interfaz de tu plataforma de datos hacia
los agentes**. Y la pregunta de diseño es exactamente la de siempre: qué expongo, con qué permisos, con qué
límites y con qué auditoría. La distinción tool/resource importa y no es cosmética: **tools son acción,
resources son contexto**. Consultar el catálogo de tablas es un resource; ejecutar una query que gasta
créditos es un tool, y por lo tanto necesita límites. Y el modelo de amenaza es real: si un agente puede
invocar una tool que ejecuta SQL arbitrario contra producción, tenés ejecución remota con esteroides. El
protocolo no te da seguridad — las políticas las implementás vos.

**Tradeoffs reales**:

| Decisión | Opción | Cuándo |
|---|---|---|
| Primitiva | Resource | Lectura de contexto: catálogo, esquemas, documentación, lineage |
| Primitiva | Tool | Acción con efecto o costo: ejecutar query, disparar pipeline |
| Alcance de la tool | SQL arbitrario | ❌ Casi nunca — superficie de ataque enorme |
| Alcance de la tool | Consultas parametrizadas y acotadas | ✅ Default: expone intención, no motor |
| Transporte | Local | Herramientas que corren en la máquina del usuario |
| Transporte | Remoto | Servidor central, multiusuario, con autenticación |
| Límites | Cuota de filas, timeout, warehouse dedicado | Siempre que la tool gaste plata |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué diferencia hay entre una tool y un resource en MCP?*
  A: Una tool es una función ejecutable: tiene efectos o costo, y el modelo decide invocarla. Un resource es dato leíble identificado por una URI, que el cliente incorpora como contexto. La separación importa porque tienen modelos de permiso distintos: leer el catálogo no es lo mismo que ejecutar algo que consume créditos.
- Q (senior): *Vas a exponer tu warehouse por MCP. ¿Cómo lo diseñás?*
  A: No expongo un ejecutor de SQL arbitrario. Expongo el catálogo y los esquemas como resources de solo lectura, y las consultas como tools parametrizadas que representan preguntas concretas del negocio, con límite de filas, timeout y un warehouse dedicado con presupuesto propio para poder cortar el gasto. Uso una identidad de servicio de solo lectura sobre las vistas que quiero exponer, no sobre todo el warehouse, y registro cada invocación con quién, qué y cuánto costó. El principio es el mismo que con cualquier API: exponer intenciones acotadas, no el motor.
- Q (trampa): *Tu servidor MCP corre local, así que la seguridad no es problema, ¿no?*
  A: Al revés: corre con los privilegios del proceso que lo lanzó, así que tiene el acceso de esa cuenta. Si expone una tool amplia y el modelo recibe una instrucción maliciosa incrustada en un documento que está leyendo, esa instrucción puede terminar invocando la tool. La defensa es la misma que en cualquier sistema: lista blanca de operaciones, validación de argumentos, mínimo privilegio y registro de auditoría.

## Lo que la doc oficial cubre bien acá

- **Google API Design Guide** (https://cloud.google.com/apis/design) — el mejor documento gratuito de diseño de APIs que existe: recursos, métodos estándar, nombres, errores, paginación. Si leés uno solo, este.
- **Microsoft REST API Guidelines** (https://github.com/microsoft/api-guidelines) — versionado, operaciones largas, paginación, con criterio corporativo.
- **Zalando RESTful API Guidelines** (https://opensource.zalando.com/restful-api-guidelines/) — nivel de detalle enorme y, lo más valioso, el razonamiento detrás de cada regla.
- **OpenAPI Specification** (https://spec.openapis.org/oas/latest.html) — la estructura formal del contrato.
- **RFC 9110** (semántica HTTP) y **RFC 9457** (Problem Details) — las fuentes autoritativas de qué significa cada verbo y cada código, y del formato estándar de errores.
- **MCP Specification** (https://modelcontextprotocol.io/specification) — primitivas, transportes, negociación de capacidades, modelo de seguridad.
- **MCP reference servers** (https://github.com/modelcontextprotocol/servers) — implementaciones reales para usar como plantilla.

## Gaps

- **Seguridad de APIs en profundidad**: las guías de diseño cubren estructura, no amenazas. Complementar con OWASP API Security Top 10 cuando el tema sea seguridad, no diseño.
- **MCP aplicado a datos**: la spec es genérica; no hay todavía una guía canónica de "cómo exponer un warehouse por MCP". Lo de este archivo es criterio derivado de principios generales de seguridad — declaralo como tal.
- **Diseño de APIs de datos específicamente** (📕 pendiente): la conexión entre modelado dimensional y contrato de salida no está bien cubierta por ninguna fuente pública única.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `rest-resource-design`: tomá tres endpoints de una API que consumís y evaluá si el verbo y el código de estado son correctos. Traeme el que esté peor.
- `api-contracts`: buscá si las APIs que consumís tienen OpenAPI publicado. Traeme cuántas sí y cuántas no. Las que no, son tu riesgo.
- `api-pagination-filtering`: revisá una ingesta tuya de API y decime cómo pagina y qué pasa si insertan filas mientras leés.
- `api-auth`: rastreá dónde vive físicamente la credencial que usa tu pipeline para su fuente principal. Traeme la respuesta honesta.
- `api-reliability`: buscá en tu código de ingesta si hay timeout y manejo de 429. Traeme el fragmento que los maneja, o la confirmación de que no existe.
- `mcp-protocol`: escribí un servidor MCP mínimo que exponga UNA consulta acotada a tu warehouse (solo lectura, con límite de filas). Traeme el código y qué límites le pusiste.

### Para subir a `mastered`

- `api-contracts`: diseñá la spec OpenAPI del contrato de salida de uno de tus productos de datos y negociala con un consumidor real. Feynman check: explicá qué cambios podés hacer sin avisar y cuáles no, y por qué.
- `api-reliability`: endurecé una ingesta real de tu empresa con timeouts, reintentos diferenciados por tipo de error y manejo de rate limit. Medí las fallas antes y después.
- `mcp-protocol`: llevá tu servidor MCP a algo que un equipo usaría, con identidad de solo lectura, cuotas y auditoría. Documentá el modelo de amenaza. Feynman check: explicá MCP a alguien de backend con una analogía que no mencione modelos de lenguaje.
- `api-pagination-filtering`: diagnosticá y corregí una ingesta que esté perdiendo o duplicando registros por paginación. Traé la evidencia de la reconciliación.

## Anti-patterns que vas a ver en clientes reales

1. **Ingesta sin timeout**
   - Cómo se hace: la llamada más simple no lo lleva y funciona en todas las pruebas.
   - Por qué se hace: nadie ve el problema hasta que la red se degrada en vez de caerse.
   - Costo real: tareas colgadas indefinidamente ocupando recursos, sin fallar ni alertar.
   - Cómo lo arregla un senior: timeout obligatorio en toda llamada de red, siempre, y un timeout a nivel de tarea como red de seguridad.

2. **Paginación por offset sobre datos vivos**
   - Cómo se hace: es lo que ofrece la API y lo que aparece en el ejemplo de la doc.
   - Por qué se hace: en pruebas con datos estáticos funciona perfecto.
   - Costo real: duplicados y salteos silenciosos. El salteo es el peor porque no deja rastro.
   - Cómo lo arregla un senior: cursor si la API lo permite; si no, merge por clave + solapamiento + reconciliación de conteos.

3. **Secreto en el repositorio o en la definición del pipeline**
   - Cómo se hace: se pone "temporalmente" para probar y queda.
   - Por qué se hace: es el camino de menor fricción y no hay un gate que lo impida.
   - Costo real: la credencial queda en el historial de Git para siempre, y rotarla implica descubrir quién la usa.
   - Cómo lo arregla un senior: almacén de secretos con resolución en ejecución, escaneo de secretos en CI, e identidad gestionada donde se pueda.

4. **Reintentar todo, incluidos los 4xx**
   - Cómo se hace: se envuelve la llamada en un reintento genérico "por las dudas".
   - Por qué se hace: parece más resiliente.
   - Costo real: se demora la detección del error real, se quema cuota, y algunos proveedores bloquean por abuso.
   - Cómo lo arregla un senior: política diferenciada — transitorios con retroceso y jitter, permanentes que fallan rápido y alertan.

5. **API sin contrato publicado**
   - Cómo se hace: el equipo proveedor documenta en una wiki, o directamente no documenta.
   - Por qué se hace: la spec se percibe como burocracia.
   - Costo real: los consumidores programan contra observaciones, y cada cambio del proveedor es un incidente sorpresa.
   - Cómo lo arregla un senior: OpenAPI generado desde el código o escrito primero, versionado en el repo, con validación de compatibilidad en el CI.

6. **Servidor MCP que expone SQL arbitrario**
   - Cómo se hace: es lo más rápido de implementar y demuestra muy bien.
   - Por qué se hace: "es solo para uso interno".
   - Costo real: superficie de ataque enorme, gasto de créditos sin control, y ninguna trazabilidad de qué se consultó.
   - Cómo lo arregla un senior: tools parametrizadas con intención acotada, identidad de solo lectura sobre vistas específicas, cuotas de filas y tiempo, warehouse dedicado con presupuesto, y auditoría de cada invocación.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés evaluar el diseño de una API ajena y decir qué está mal y por qué, con argumentos de interoperabilidad?
- [ ] ¿Podés distinguir un cambio compatible de uno que rompe, incluyendo los casos grises?
- [ ] ¿Podés explicar por qué offset falla con escrituras concurrentes, y qué hacer si no podés cambiarlo?
- [ ] ¿Podés diseñar la gestión de credenciales de una plataforma sin que ningún secreto viva en un repo?
- [ ] ¿Podés escribir una política de reintentos que distinga transitorio de permanente y explicar el jitter?
- [ ] ¿Podés diseñar un servidor MCP sobre tu warehouse con su modelo de amenaza explícito?
- [ ] En entrevista senior, ¿podés contestar "la ingesta trae datos duplicados a veces" con un diagnóstico ordenado en vez de agregar un `DISTINCT`?
