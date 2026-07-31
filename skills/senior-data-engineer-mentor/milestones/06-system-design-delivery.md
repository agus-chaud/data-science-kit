# Hito 6 — System Design & Delivery

## Por qué importa (perspectiva corporativa)

Este es el hito que te convierte en arquitecto. Hasta acá aprendiste herramientas; este hito es sobre
**cómo encajan las piezas y cómo llega el cambio a producción sin romper nada**.

Y hay algo que decir de frente: la mayoría de los data engineers se quedan del lado del dato y tratan al
backend y al frontend como "el otro equipo". Ese muro es exactamente lo que te frena en el nivel medio.
Porque las decisiones que más duelen viven en la frontera: dónde vive la lógica de negocio, quién es dueño
del contrato, si el front puede consultar el warehouse (no, no puede — y tenés que poder explicar por qué
con argumentos, no con dogma), cómo llega un cambio de modelo a producción sin dejar un dashboard roto.

Del lado de la entrega: **CI/CD para datos no es CI/CD para software, y ahí se equivoca todo el mundo**. En
software, correr todos los tests en cada PR es lo correcto. En datos, correr todo el proyecto en cada PR te
sale carísimo y encima es lento. La solución existe y es específica del dominio — construir solo lo que
cambió y lo que depende de eso — y saberla te distingue rápido.

## Conceptos de este hito

### backend-frontend-split

**Qué es**: El reparto de responsabilidades. El **frontend** presenta e interactúa. El **backend** contiene
la lógica de negocio, valida, autoriza y accede a datos. El patrón **BFF (Backend For Frontend)** agrega una
capa por tipo de cliente, que adapta y compone datos a la medida de esa interfaz.

**La trampa del junior** (y también del que viene de datos): querer que el front consulte directamente el
warehouse "porque el dato ya está ahí". Se rompe todo: credenciales del warehouse expuestas del lado del
cliente, autorización por fila imposible de garantizar, latencia analítica en una interfaz interactiva, y
costo de créditos disparado por cada usuario que refresca la pantalla.

**Cómo lo piensa un senior**: la frontera correcta se decide con tres preguntas. *¿Quién autoriza?* Siempre
el backend — cualquier validación en el front es cosmética, porque el cliente se puede modificar. *¿Dónde
vive la regla de negocio?* En un solo lugar, y si se duplica en el front por experiencia de usuario, el
backend sigue siendo la autoridad. *¿Qué latencia tolera esta pantalla?* Eso determina si el dato puede
venir de un sistema analítico o necesita una capa de servicio. El BFF entra cuando distintos clientes
necesitan formas distintas del mismo dato y no querés contaminar el servicio de dominio con las necesidades
de cada pantalla.

**Tradeoffs reales**:

| Arquitectura | Cuándo | Contra |
|---|---|---|
| Front → API de dominio directo | Un solo cliente, necesidades alineadas | Cada cambio de UI presiona al servicio de dominio |
| Front → BFF → servicios | Varios clientes (web, móvil) con formas distintas | Una capa más que mantener y desplegar |
| Front → warehouse directo | ❌ Prácticamente nunca | Credenciales expuestas, sin autorización fina, costo y latencia |
| Front → base de servicio (réplica del warehouse) | Necesitás latencia baja sobre datos analíticos | Un pipeline más que sincronizar y monitorear |
| GraphQL como capa de composición | Clientes muy heterogéneos | Complejidad de caché, control de costo de queries |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es el patrón BFF y qué problema resuelve?*
  A: Una capa de backend dedicada a un tipo de cliente, que compone y adapta datos de varios servicios a la forma exacta que esa interfaz necesita. Resuelve que web y móvil tengan necesidades distintas sin ensuciar los servicios de dominio con lógica de presentación ni obligar al cliente a hacer cinco llamadas.
- Q (senior): *El equipo de producto quiere que el dashboard consulte Snowflake directo para "ahorrar desarrollo". ¿Qué respondés?*
  A: Que resuelve hoy y genera cuatro problemas. Credenciales del warehouse del lado del cliente, con acceso más amplio que el que ese usuario debería tener. Autorización a nivel de fila que no se puede garantizar desde el front. Latencia analítica en una pantalla interactiva. Y costo por usuario, sin control, porque cada refresco enciende compute. La alternativa concreta es una API delgada que sirva las métricas ya agregadas, con caché y autorización en el backend — y si la latencia sigue sin alcanzar, una tabla de servicio precalculada. El costo de desarrollo que se ahorra hoy vuelve multiplicado en la primera auditoría de seguridad.
- Q (trampa): *Si el front ya valida el formulario, ¿el backend puede confiar?*
  A: Nunca. La validación del front es experiencia de usuario: evita un viaje al servidor y da feedback rápido. Pero el cliente es código que corre en una máquina que no controlás y las peticiones se pueden fabricar a mano. La validación real, la de verdad, es la del backend. Duplicarla es correcto; delegarla, no.

### data-serving-layer

**Qué es**: Cómo sale el dato del warehouse hacia sus consumidores. Opciones: **API de datos** sobre el
warehouse, **base de servicio** (copia optimizada para lectura de baja latencia), **reverse ETL** (empujar
datos del warehouse a herramientas operativas), **semantic layer** (definiciones de métricas centralizadas)
o consumo directo por herramientas de BI.

**La trampa del junior**: exponer el warehouse a todo el mundo y llamarlo "democratización". Sin capa de
servicio, cada consumidor define sus propias métricas con su propio SQL, y tres áreas llegan a la reunión con
tres números distintos de "ventas del mes". Además, cada herramienta que se conecta consume compute sin
control.

**Cómo lo piensa un senior**: **el warehouse es un motor de cómputo, no un servidor de aplicaciones**. Está
optimizado para escanear mucho, no para responder miles de consultas chicas con latencia de milisegundos. La
capa de servicio se elige por la combinación de latencia requerida, concurrencia esperada y quién define la
métrica. Y ese último punto es el más importante y el menos técnico: si la definición de una métrica vive en
cada dashboard, vas a tener tantas verdades como dashboards. El semantic layer existe para que la definición
tenga un solo dueño.

**Tradeoffs reales**:

| Patrón | Latencia | Concurrencia | Cuándo |
|---|---|---|---|
| BI conectado al warehouse | Segundos | Baja-media | Análisis exploratorio, reportes |
| API sobre el warehouse con caché | Cientos de ms a segundos | Media | Dashboards internos, integraciones |
| Base de servicio (Postgres/Redis) alimentada por pipeline | Milisegundos | Alta | Producto de cara al usuario |
| Reverse ETL | Minutos | — | Llevar segmentos al CRM, a marketing |
| Semantic layer | — | — | Cuando la definición de métrica se está duplicando |
| Acceso directo al warehouse desde la app | ❌ | ❌ | Prácticamente nunca |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué no exponer el warehouse directamente a una aplicación?*
  A: Porque está diseñado para consultas analíticas grandes y poco frecuentes, no para muchas consultas chicas con latencia baja. Además el costo escala con el uso de la aplicación, y el control de acceso a nivel de fila y de usuario final es mucho más difícil de garantizar desde ahí.
- Q (senior): *Tres áreas reportan tres números distintos de la misma métrica. ¿Cómo lo resolvés?*
  A: El problema no es técnico primero, es de propiedad: hay que definir quién es dueño de esa métrica y cuál es su definición canónica. Con eso acordado, lo técnico es centralizarla — en un mart con el grano declarado o en un semantic layer — y migrar los tres consumidores a esa fuente, deprecando los cálculos locales. Y para que no vuelva a pasar, la métrica se define una vez y se consume, nunca se recalcula en cada herramienta.
- Q (trampa): *¿Reverse ETL es lo mismo que una API de datos?*
  A: No. Reverse ETL empuja datos del warehouse hacia sistemas operativos de forma programada — el consumidor no consulta, recibe. Una API es sincrónica y bajo demanda. Se eligen por el patrón de consumo: si la herramienta destino necesita el dato adentro para operar (el CRM con el segmento del cliente), es reverse ETL; si alguien pregunta cuando lo necesita, es API.

### azure-pipelines

**Qué es**: El motor de CI/CD de Azure DevOps, definido en YAML: **stages** que contienen **jobs** que
contienen **steps**. Se conecta a recursos externos con **service connections**, guarda configuración en
**variable groups** (con enlace a Key Vault), reutiliza definiciones con **templates**, y controla la
promoción con **environments** que llevan aprobaciones y verificaciones.

**La trampa del junior**: pelearse durante horas con las tres sintaxis de expresión sin saber que son tres
cosas distintas que se resuelven en momentos distintos. Una se evalúa cuando se compila la plantilla, otra
cuando corre el pipeline, y la tercera es sustitución textual antes de ejecutar el comando. Confundirlas
produce errores incomprensibles del tipo "la variable está vacía y no entiendo por qué".

**Cómo lo piensa un senior**: **el pipeline es código de infraestructura, no un script**. Se versiona, se
revisa y se reutiliza. Los templates no son solo para no repetir: son el mecanismo de **gobierno** — una
plantilla que todos los pipelines deben extender es donde imponés los pasos obligatorios de seguridad y
calidad, sin depender de que cada equipo se acuerde. Y los environments con aprobaciones son donde vive la
separación real entre "el código está listo" y "el cambio está autorizado a entrar a producción".

**Tradeoffs reales**:

| Mecanismo | Para qué | Cuidado |
|---|---|---|
| Expresión de plantilla (compile-time) | Decidir qué pasos existen | No ve valores que se calculan durante la corrida |
| Expresión de runtime | Condiciones según resultados | No puede crear ni eliminar pasos |
| Sustitución de macro | Pasar valores a comandos | Si la variable no existe, queda el literal — falla silenciosa |
| Variable group + Key Vault | Secretos y configuración | Los secretos no se enmascaran solos en todos los contextos |
| Service connection | Autenticación contra la nube | Con federación de credenciales evitás secretos rotables |
| Environment con aprobación | Gate humano a producción | Si aprueba siempre la misma persona, es un trámite, no un control |
| Template obligatoria (`extends`) | Gobierno de seguridad | Requiere disciplina organizacional |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es una service connection y por qué no se ponen credenciales en el YAML?*
  A: Es la definición de acceso a un recurso externo, guardada y protegida por el proyecto, con permisos sobre quién puede usarla. Se usa para que el YAML — que vive en el repositorio y lo lee cualquiera con acceso — nunca contenga credenciales.
- Q (senior): *¿Cómo garantizás que todos los pipelines de la organización corran los chequeos de seguridad?*
  A: Con una plantilla obligatoria que los pipelines extienden. Al extender, el pipeline hereda los pasos definidos por la plantilla y no puede saltearlos, así que el control deja de depender de que cada equipo lo agregue. Eso se complementa con políticas de rama que exigen que el pipeline pase antes de mergear, y con permisos sobre las service connections de producción para que solo pipelines autorizados puedan usarlas.
- Q (trampa): *Una variable aparece vacía en un paso. ¿Por dónde empezás?*
  A: Por identificar en qué momento se resuelve. Si usé una expresión de plantilla para algo que solo se conoce durante la ejecución, se evaluó antes de que el valor existiera y quedó vacía. Si usé sustitución de macro y la variable no está definida, queda el texto literal en vez del valor. La mayoría de estos problemas no son de valor sino de momento de evaluación.

### cicd-for-data

**Qué es**: Aplicar integración y entrega continua a un proyecto de datos: validar los cambios de modelos en
una rama antes de mergear, y promoverlos por ambientes. La técnica clave es construir **solo lo que cambió y
lo que depende de eso** (`state:modified+`), difiriendo el resto a producción en lugar de reconstruirlo.

**La trampa del junior**: correr el proyecto entero en cada pull request. Es lo que uno hace por analogía con
el software, y en datos significa reconstruir tablas de miles de millones de filas para validar un cambio de
tres líneas. Sale carísimo, tarda una eternidad y termina con el equipo desactivando el CI.

**Cómo lo piensa un senior**: el CI de datos tiene que responder **"¿este cambio rompe algo?"** en minutos y
por poca plata. Y eso se logra con dos ideas: comparar contra un manifiesto del estado de producción para
saber qué cambió realmente, y diferir todo lo no modificado a los objetos productivos existentes en vez de
reconstruirlos. Sobre ambientes: la separación mínima real es que cada desarrollador escriba en su propio
esquema, que el CI escriba en un esquema efímero de la rama, y que producción sea el único lugar donde
escribe el job programado. Y una advertencia que suena obvia y se viola todo el tiempo: **el CI tiene que
correr con datos representativos**. Un CI sobre una muestra de mil filas pasa verde ante problemas que solo
aparecen con volumen o con casos raros.

**Tradeoffs reales**:

| Estrategia de CI | Costo | Confianza | Cuándo |
|---|---|---|---|
| Correr todo el proyecto | Muy alto | Alta | Proyectos chicos únicamente |
| Solo modificados + descendientes, con deferral | Bajo | Alta | Default para cualquier proyecto real |
| Solo compilar y lint | Mínimo | Baja | Puerta rápida, insuficiente sola |
| Correr sobre muestra reducida | Bajo | Media-baja | Complemento, nunca reemplazo |
| Ambiente de staging con copia de producción | Medio | Muy alta | Cambios de alto riesgo, migraciones |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué no correr todo el proyecto dbt en cada PR?*
  A: Porque reconstruir todos los modelos cuesta compute y tiempo desproporcionados frente al cambio que se está validando. Como dbt conoce el DAG, se puede construir solo lo modificado y sus descendientes, y tomar el resto desde producción sin reconstruirlo.
- Q (senior): *¿Cómo diseñás la promoción de cambios de dev a producción en una plataforma de datos?*
  A: Cada desarrollador con su esquema propio, para que nadie pise a nadie. En el pull request, un job que construye solo lo modificado y sus descendientes en un esquema efímero de la rama y corre sus tests; si pasa, el esquema se destruye. Al mergear, el despliegue actualiza producción, y los cambios de alto riesgo — renombres de columnas, cambios de grano — pasan por un paso extra con aprobación y un plan de reversión. La clave es que cada ambiente tenga un dueño claro de escritura: si el job programado y una persona pueden escribir la misma tabla, no hay ambiente, hay caos compartido.
- Q (trampa): *Tu CI pasa verde siempre. ¿Buena señal?*
  A: Sospechosa. Un CI que nunca falla suele estar corriendo sobre datos que no son representativos, o teniendo tests que no verifican nada sustancial, o excluyendo justamente los modelos pesados donde están los problemas. Un CI útil falla de vez en cuando: esa es la evidencia de que está mirando algo.

### iac-secrets

**Qué es**: Gestionar infraestructura y credenciales como código y como configuración externa: **Key Vault**
para secretos, **variable groups** para configuración por ambiente, **service principals** o identidades
federadas para autenticación de pipelines, e **infraestructura como código** para que los recursos sean
reproducibles.

**La trampa del junior**: el archivo `.env` con la contraseña de Snowflake commiteado. O la credencial
pegada en la conexión de Airflow y nunca rotada, porque rotarla implica descubrir quién la usa.

**Cómo lo piensa un senior**: **si un secreto entró al repositorio, ya está comprometido** — el historial de
Git es para siempre y borrar el archivo no lo borra. De ahí salen dos reglas duras: escaneo de secretos como
gate del CI, y resolución en tiempo de ejecución desde el almacén. Y la jugada superior es eliminar el
secreto: con identidad federada, el pipeline se autentica con su identidad del sistema de CI y no hay nada
que rotar ni que filtrar. Sobre infraestructura como código, el argumento fuerte no es la automatización — es
que el estado de producción sea **auditable y reproducible**: podés responder qué cambió, cuándo y quién lo
aprobó, que es la pregunta que hace toda auditoría.

**Tradeoffs reales**:

| Práctica | Beneficio | Costo |
|---|---|---|
| Secreto en almacén, resuelto en ejecución | Rotación centralizada, auditoría de acceso | Una dependencia más en el arranque |
| Identidad federada / gestionada | No hay secreto que rotar ni filtrar | Requiere soporte del proveedor y configuración inicial |
| Escaneo de secretos en CI | Corta el problema en el origen | Falsos positivos al principio |
| IaC completo | Reproducible, auditable, revisable | Curva de aprendizaje, deriva si alguien toca a mano |
| Configuración manual en la consola | Rápido hoy | Nadie sabe por qué está así en seis meses |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué no alcanza con borrar un secreto commiteado?*
  A: Porque queda en el historial del repositorio y en todos los clones existentes. Cualquiera con acceso al historial lo recupera. La única respuesta correcta es rotar el secreto, no borrarlo del código.
- Q (senior): *¿Cómo eliminás secretos de larga vida de una plataforma?*
  A: Reemplazándolos por identidad en vez de credencial. Donde el proveedor lo permita, identidad gestionada o federación desde el sistema de CI: el pipeline prueba quién es y recibe un token efímero, así que no hay nada persistente que robar ni rotar. Para lo que no lo soporte, secreto en el almacén con rotación automatizada y una identidad distinta por carga de trabajo, para poder revocar sin afectar a todos. Y un escaneo en el CI que impida que vuelva a entrar uno al repo.
- Q (trampa): *La variable está marcada como secreta en el pipeline, así que no se puede filtrar, ¿no?*
  A: El enmascarado ayuda pero no es una garantía. Si el valor pasa por una transformación, se codifica, o se imprime en partes, puede aparecer en los logs sin que el enmascarado lo detecte. Y cualquiera que pueda modificar el pipeline puede agregar un paso que lo exfiltre. El enmascarado protege del accidente, no del acceso.

### data-governance-cost

**Qué es**: Que la plataforma sea gobernable: **RBAC** con roles por función, **lineage** para responder de
dónde viene un dato, **data contracts** entre productores y consumidores, **SLA/SLO de datos** (frescura,
completitud, disponibilidad) y **atribución de costo** por consumidor o dominio.

**La trampa del junior**: pensar que gobierno es burocracia que frena. Hasta que un dashboard queda
desactualizado tres días y nadie se entera, o alguien borra una tabla sin saber que alimentaba el reporte del
directorio, o la factura sube 40% y no hay forma de saber de quién es.

**Cómo lo piensa un senior**: gobierno es **poder responder preguntas sobre tu propia plataforma**: quién
accede a qué, de dónde viene este número, quién se rompe si cambio esto, quién gastó esto. Si no podés
responderlas, no tenés plataforma, tenés un conjunto de pipelines. Y el SLO de datos es la herramienta más
subestimada: comprometerse formalmente a "este mart está actualizado hasta las 8 AM el 99% de los días
hábiles" convierte una expectativa difusa en algo medible y alertable. La regla que más rinde: **el equipo de
datos tiene que enterarse antes que el negocio, siempre**. Si el negocio te avisa que un dato está viejo,
perdiste — no el dato, la confianza.

**Tradeoffs reales**:

| Práctica | Beneficio | Costo |
|---|---|---|
| RBAC por rol funcional | Permisos auditables y escalables | Diseño inicial y mantenimiento |
| Permisos por persona | Rápido al principio | Inmanejable a los seis meses |
| Lineage automático (dbt + catálogo) | Análisis de impacto real antes de cambiar | Requiere que todo pase por `ref()` |
| Data contracts con consumidores | Cambios sin sorpresas | Reduce tu libertad de cambio unilateral |
| SLO de datos con alertas | Te enterás antes que el negocio | Hay que definirlos y sostenerlos |
| Atribución de costo por dominio | Conversación de costo con dueño | Requiere separar warehouses y etiquetar |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es un SLO de datos y en qué se diferencia de un test?*
  A: Un test verifica una propiedad de los datos cuando corre el pipeline. Un SLO es un compromiso medible sobre el servicio en el tiempo — frescura, completitud, disponibilidad — que se monitorea de forma continua y dispara alertas cuando se incumple, corra o no el pipeline. Un pipeline que no corrió pasa todos sus tests y viola el SLO de frescura.
- Q (senior): *¿Cómo hacés análisis de impacto antes de cambiar el grano de un mart?*
  A: Desde el lineage saco los modelos descendientes, y del log de consultas del warehouse saco los consumidores reales de esa tabla en los últimos meses — que casi nunca coinciden con los que el lineage muestra, porque hay dashboards y notebooks que consultan directo. Con esa lista contacto a los dueños, acuerdo una ventana, y si hay contrato de por medio, versiono en vez de romper. El cambio se comunica antes, no después.
- Q (trampa): *Tenés lineage completo en dbt. ¿Ya sabés quién se rompe si cambiás una tabla?*
  A: Sabés quién se rompe **dentro** de dbt. Los consumidores de afuera — dashboards conectados directo, notebooks, exportaciones, otras aplicaciones — no aparecen en ese grafo. Esos se descubren en el log de consultas del warehouse, y son justamente los que se enteran por sorpresa.

## Lo que la doc oficial cubre bien acá

- **Azure Architecture Center — Cloud Design Patterns** (https://learn.microsoft.com/azure/architecture/patterns/) — catálogo de patrones con tradeoffs explícitos, incluido BFF. Aplicable fuera de Azure.
- **Backends for Frontends** (https://learn.microsoft.com/azure/architecture/patterns/backends-for-frontends) — el patrón con sus contras.
- **Azure Well-Architected Framework** (https://learn.microsoft.com/azure/well-architected/) — los cinco pilares. Es el vocabulario para defender decisiones ante management.
- **Azure Pipelines YAML schema** (https://learn.microsoft.com/azure/devops/pipelines/yaml-schema/) y **Expressions** (https://learn.microsoft.com/azure/devops/pipelines/process/expressions) — la referencia exacta, incluida la distinción entre momentos de evaluación.
- **dbt node selection** (https://docs.getdbt.com/reference/node-selection/syntax) — `state:modified`, `--defer`, selectores. Es la base técnica del CI barato.
- **Snowflake access control** (https://docs.snowflake.com/en/user-guide/security-access-control-overview) — jerarquía de roles y buenas prácticas de RBAC.
- **martinfowler.com** — BFF, integración continua, patrones de arquitectura. Es la fuente canónica de vocabulario.

## Gaps

- **Arquitectura de sistemas distribuidos** (📕 pendiente): consistencia, replicación, particionamiento están en *DDIA* (Kleppmann). Sustituto parcial: el catálogo de patrones de Azure y el SRE Book de Google (gratuito).
- **Organización de equipos de datos**: data mesh, ownership por dominio, contratos organizacionales. No hay fuente oficial única; lo que hay son artículos de práctica. Declaralo cuando el tema salga.
- **FinOps aplicado a datos**: el marco general existe pero la bajada a warehouses está dispersa. Lo concreto de Snowflake está en el Hito 2.

## Ejercicios para subir de nivel

### Para subir a `practiced` (el gimnasio es tu laburo)

- `backend-frontend-split`: dibujá quién llama a quién en un producto de tu empresa, desde el front hasta el warehouse. Marcá dónde vive la lógica de negocio. Traeme el dibujo.
- `data-serving-layer`: identificá cómo llega el dato desde el warehouse hasta el usuario final en tu empresa y qué latencia tolera. Traeme el camino completo.
- `azure-pipelines`: leé un pipeline YAML de tu repo y marcá cuál es su gate a producción y quién aprueba. Traeme la respuesta.
- `cicd-for-data`: averiguá si tu CI corre todo el proyecto dbt o solo lo modificado. Traeme cuánto tarda y, si podés, cuánto cuesta por PR.
- `iac-secrets`: rastreá dónde está guardada la credencial de Snowflake que usa tu orquestador y cuándo se rotó por última vez. Traeme la respuesta honesta.
- `data-governance-cost`: preguntate quién se entera primero si un mart queda desactualizado. Si la respuesta es el negocio, ese es tu hallazgo.

### Para subir a `mastered`

- `cicd-for-data`: implementá o mejorá el CI de datos de tu equipo para que construya solo lo modificado. Medí tiempo y costo antes y después. Feynman check: explicá por qué el CI de datos no puede copiar al de software.
- `data-governance-cost`: definí el SLO de frescura de un mart real, instrumentá la alerta y sostenela un mes. Traé cuántas veces alertó y si el negocio se enteró antes o después que vos.
- `backend-frontend-split`: escribí y defendé la propuesta de una capa de servicio para un caso donde hoy se consulta el warehouse directo, con costo y latencia de cada opción.
- `iac-secrets`: eliminá un secreto de larga vida de tu plataforma reemplazándolo por identidad. Documentá el antes y el después.

## Anti-patterns que vas a ver en clientes reales

1. **El front consultando el warehouse**
   - Cómo se hace: alguien lo hizo para un prototipo, funcionó, y quedó en producción.
   - Por qué se hace: ahorra desarrollo hoy y el dato "ya está ahí".
   - Costo real: credenciales expuestas, sin autorización por fila, latencia mala y costo que escala con los usuarios.
   - Cómo lo arregla un senior: API delgada con las métricas agregadas, caché y autorización en el backend. Si la latencia no alcanza, tabla de servicio precalculada.

2. **CI que corre el proyecto entero**
   - Cómo se hace: por analogía con el CI de software.
   - Por qué se hace: es lo correcto en software y nadie cuestiona la analogía.
   - Costo real: PRs que tardan una hora y cuestan una fortuna. Termina en que el equipo desactiva el CI, que es el peor final posible.
   - Cómo lo arregla un senior: construir modificados y descendientes con deferral a producción, en un esquema efímero por rama.

3. **Un warehouse compartido sin atribución**
   - Cómo se hace: se creó uno al principio y todos lo usan.
   - Por qué se hace: nunca hubo un momento explícito de decidir otra cosa.
   - Costo real: el costo no tiene dueño, así que no hay ninguna conversación posible sobre optimizarlo.
   - Cómo lo arregla un senior: separar por dominio o equipo, etiquetar, y reportar el costo a cada dueño. La atribución es la que habilita la conversación.

4. **Aprobaciones que son un trámite**
   - Cómo se hace: se configura un environment con aprobación y siempre aprueba la misma persona sin mirar.
   - Por qué se hace: cumple el requisito de auditoría con el mínimo esfuerzo.
   - Costo real: el control existe en el papel y no en los hechos, y da una falsa sensación de seguridad que hace que nadie mire nada más.
   - Cómo lo arregla un senior: la aprobación tiene que tener contenido — el pipeline muestra qué modelos cambian y qué consumidores afecta, para que aprobar signifique algo.

5. **Nadie sabe quién consume qué**
   - Cómo se hace: se crean tablas, la gente se conecta, y nunca hay un registro.
   - Por qué se hace: registrar consumidores requiere un proceso y nadie lo instaló.
   - Costo real: cualquier cambio es una ruleta rusa, así que en la práctica nada se cambia y la deuda se acumula.
   - Cómo lo arregla un senior: lineage para lo interno, log de consultas del warehouse para lo externo, y contratos explícitos para los consumidores críticos.

6. **Secretos rotados nunca**
   - Cómo se hace: se configuró la credencial una vez y funciona.
   - Por qué se hace: rotarla implica saber quién la usa, y nadie sabe.
   - Costo real: una credencial de larga vida con permisos amplios es exactamente lo que un atacante busca.
   - Cómo lo arregla un senior: identidad por carga de trabajo, rotación automatizada, y donde se pueda, identidad federada para que no haya secreto.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés explicar con argumentos —no con dogma— por qué el front no consulta el warehouse, y proponer la alternativa concreta?
- [ ] ¿Podés elegir capa de servicio a partir de latencia, concurrencia y propiedad de la métrica?
- [ ] ¿Podés distinguir los momentos de evaluación de un pipeline YAML y debuggear una variable vacía?
- [ ] ¿Podés diseñar un CI de datos que valide de verdad sin reconstruir el proyecto entero?
- [ ] ¿Podés diseñar la gestión de credenciales de una plataforma sin secretos de larga vida?
- [ ] ¿Podés hacer análisis de impacto real, incluyendo los consumidores que el lineage no ve?
- [ ] En entrevista senior, ¿podés contestar "cómo llevás un cambio de modelo a producción" con un proceso completo en vez de "hago merge"?
