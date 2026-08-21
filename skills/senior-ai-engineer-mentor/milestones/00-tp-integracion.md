# Hito 0 — Integración práctica (webhook + agente sobre una plataforma real)

## Por qué importa (perspectiva corporativa)

Este es el hito que separa a alguien que sabe usar un SDK de LLM de alguien que puede meter un agente en un sistema real ajeno — WhatsApp, Slack, un CRM, lo que sea. La diferencia no está en el LLM, está en todo lo que lo RODEA: cómo te enteras de que pasó algo (webhook), de dónde saca memoria el agente cuando vos no la controlás (historial de un tercero), y cómo evitás que una alucinación del modelo se convierta en un dato falso guardado en tu base.

Un AI Engineer que solo sabe llamar `client.messages.create()` en un notebook no está listo para producción. Producción es: un proveedor externo (Kapso, Twilio, Slack) te empuja eventos por HTTP, vos corrés en una función serverless sin estado propio, y tenés que decidir en tiempo real qué hacer — sin poder confiar ciegamente en lo que el LLM te dice que hizo.

Las oportunidades que abre este hito: cualquier posición de "integraciones" o "platform engineer" con foco en AI, roles donde el trabajo es "conectar el LLM a X sistema legado", y — más importante — la capacidad de auditar en una entrevista si un candidato realmente construyó algo end-to-end o solo copió un ejemplo de la doc.

## Conceptos de este hito

### webhook-vs-polling

**Qué es**: dos formas de enterarte de que algo pasó en un sistema externo. Polling: le preguntás cada tanto "¿hay algo nuevo?". Webhook: el sistema externo te avisa solo, con un `POST` HTTP a una URL tuya, apenas pasa el evento.

**La trampa del junior**: asumir que un webhook es "más simple" porque no hay que armar un loop de polling. Al revés — un webhook te obliga a tener una URL pública, siempre disponible, corriendo SIN estado entre invocaciones (cada request puede ser una instancia nueva). El junior arma la función asumiendo que puede guardar algo "en memoria" entre un webhook y el siguiente, y se sorprende cuando desaparece.

**Cómo lo piensa un senior**: un webhook es un contrato de disponibilidad — si tu endpoint está caído cuando llega el evento, ¿el proveedor reintenta? ¿cuántas veces? ¿con qué backoff? Un senior lee la doc de reintentos del proveedor ANTES de asumir que "si falla, ya fue". También diseña la función para ser **idempotente**: si el mismo evento llega dos veces (reintentos duplicados son comunes), procesarlo dos veces no debe romper nada.

**Tradeoffs reales**:

| Approach | Latencia | Costo | Complejidad |
|---|---|---|---|
| Polling | Alta (depende del intervalo) | Alto si el intervalo es corto | Baja — no necesitás URL pública |
| Webhook | Baja (push inmediato) | Bajo — solo procesás eventos reales | Media — necesitás endpoint público + idempotencia |
| Webhook + cola (SQS/pubsub) | Baja, con buffer | Medio | Alta — pero aguanta picos sin perder eventos |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué preferís webhook sobre polling para eventos de mensajería?*
  A: Latencia y costo. Polling cada 5 segundos son 17,280 requests/día por conexión, la mayoría vacíos. Webhook solo te cuesta cuando hay evento real.
- Q (senior): *Tu endpoint de webhook estuvo caído 3 minutos. ¿Perdiste eventos?*
  A: Depende del proveedor. Hay que revisar su política de reintentos (exponential backoff, cuántos intentos, por cuánto tiempo) ANTES de asumir garantías que no existen. Si el proveedor no reintenta, necesitás un mecanismo propio (polling de respaldo, o una cola intermedia).
- Q (trampa): *Tu función de webhook procesó el mismo mensaje dos veces y mandó una respuesta duplicada. ¿De quién es el bug?*
  A: Tuyo, no del proveedor. Los reintentos con duplicados son esperables — la función tiene que ser idempotente (chequear si ya procesaste ese `message_id` antes de actuar) o al menos tolerante a duplicados sin efectos visibles para el usuario final.

### agent-conversation-memory

**Qué es**: cuando la memoria del agente no vive en un store que vos diseñaste, sino en el historial que la PLATAFORMA externa ya guarda (el historial de conversación de WhatsApp/Slack/lo que sea). En vez de armar tu propia tabla de "estado de la conversación", le pasás al LLM el historial real como contexto en cada turno.

**La trampa del junior**: reinventar el estado a mano — una tabla propia de "en qué paso está esta conversación" — cuando la plataforma ya te da esa información gratis. Además, asumir que el formato en que la plataforma te devuelve mensajes VIEJOS es neutro: si ese formato tiene artefactos propios de logging/UI, el LLM los puede imitar como si fueran parte de su forma natural de responder.

**Cómo lo piensa un senior**: la memoria de un agente no es pasiva — el modelo aprende ESTILO además de CONTENIDO de lo que ve en su propio historial. Antes de pasarle mensajes previos como contexto, un senior se pregunta: ¿este texto es información limpia, o trae formato de una capa de UI/logging que no debería imitarse? Cura el historial (normaliza, resume, o reemplaza artefactos de formato) antes de dárselo al modelo como si fuera "lo que yo mismo dije antes".

**Tradeoffs reales**:

| Approach | Consistencia de estado | Esfuerzo de implementación | Riesgo |
|---|---|---|---|
| Tabla de estado propia (FSM) | Total, 100% controlada | Alto — hay que mantenerla sincronizada | Ninguno de imitación, pero reinventa algo que ya existe |
| Historial de la plataforma, crudo | Depende de la plataforma | Bajo | El modelo puede imitar artefactos de formato del historial |
| Historial curado (normalizado) | Depende de la plataforma | Medio | Bajo, si la curación es correcta |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué usar el historial de la plataforma en vez de armar tu propia tabla de estado?*
  A: Evita duplicar una fuente de verdad. Si la plataforma ya trackea la conversación, mantener tu propia copia sincronizada es trabajo extra que puede desincronizarse.
- Q (senior): *Tu agente empezó a responder con un formato raro que vos nunca programaste. ¿Por dónde empezás a debuggear?*
  A: Por el historial que le estás pasando como contexto. Si algún mensaje previo (sobre todo generado por tu propio sistema, no por el usuario) tiene un formato inusual, el modelo lo puede estar copiando como "así hablo yo".
- Q (trampa): *¿Confiarías en que el LLM transcriba EXACTO un ID que vio en el historial (una fecha, un código)?*
  A: No sin validar. Los modelos pueden cometer errores de transcripción al copiar strings literales de un contexto largo — hay que validar server-side contra el conjunto de valores válidos antes de cualquier efecto con consecuencias (escritura en base, cobro, etc.).

### tool-output-validation

**Qué es**: verificar, en TU código (no en el prompt), que los argumentos con los que el LLM llamó una tool son válidos ANTES de ejecutar el efecto (insertar en base, cobrar, enviar algo irreversible) — incluso cuando el system prompt ya le pidió al modelo que solo use valores válidos.

**La trampa del junior**: confiar en que "si el prompt lo dice, el modelo lo va a respetar". Un system prompt es una instrucción, no una garantía — el modelo puede desviarse (interpretar texto libre cuando no debía, transcribir mal un valor, o directamente responder en prosa sin llamar ninguna tool) y nada te avisa salvo que vos lo chequees.

**Cómo lo piensa un senior**: aplica el mismo principio de "nunca confíes en el input" que usarías con un formulario público — la diferencia es que acá el "usuario" que manda el input es el LLM. Valida cada argumento contra el conjunto de valores realmente válidos (una lista cerrada, un rango, un formato) antes de ejecutar cualquier efecto con consecuencias. Si algo no valida, no ejecutes — devolvé un mensaje claro y, si aplica, reintentá la interacción.

**Tradeoffs reales**:

| Approach | Garantía | Costo de implementación | Cuándo alcanza |
|---|---|---|---|
| Solo instrucción en el prompt | Ninguna real — es una sugerencia | Cero | Nunca, para efectos con consecuencias reales |
| Validación server-side contra valores conocidos | Alta — bloquea el caso malo antes del efecto | Bajo-medio | Casi siempre que hay un conjunto cerrado de valores válidos |
| `tool_choice` forzado (obligar a usar una tool específica) | Garantiza que SE LLAMA una tool, no que los args son válidos | Bajo | Cuando no hace falta permitir respuesta libre en ese punto del flujo |
| Constraint a nivel de base de datos (UNIQUE, CHECK, FK) | Total, para lo que la base puede expresar | Bajo, una vez | Siempre que aplique — es la última línea de defensa |

**En entrevista te van a preguntar**:
- Q (mid): *El system prompt dice "nunca inventes esta fecha". ¿Alcanza eso?*
  A: No. Es una instrucción de comportamiento, no una garantía técnica. Hay que validar el valor recibido contra el conjunto de fechas realmente ofrecidas antes de usarlo en cualquier efecto.
- Q (senior): *¿Dónde ponés la validación: en el prompt, en el código de la tool, o en la base de datos?*
  A: En las tres capas, con roles distintos — "defense in depth". El prompt guía el comportamiento esperado (barato, pero no confiable solo). El código de la tool valida antes de actuar (la garantía real). La base de datos es la última línea (constraints) para lo que sea expresable ahí — protege incluso si el código de la tool tiene un bug.
- Q (trampa): *Tu agente le dijo al usuario "listo, guardado" pero la base está vacía. ¿Cómo lo detectás y cómo lo evitás?*
  A: Se detecta comparando lo que el agente DICE con lo que realmente se ejecutó (logs de la tool, no el texto de la respuesta). Se evita separando estrictamente "el modelo generó texto" de "se ejecutó una tool con éxito" — nunca asumas que un mensaje de confirmación implica que el efecto ocurrió; confirmá con el propio resultado de la ejecución.

### external-platform-auth-patterns

**Qué es**: los 3 mecanismos típicos para autenticarte contra una plataforma externa — **API key** (credencial fija, para procesos desatendidos), **sesión de usuario vía CLI/OAuth** (un humano se autentica una vez, la sesión queda guardada), y **MCP** (protocolo que le da a un LLM acceso a herramientas de un servicio, autenticado por dentro con alguno de los dos anteriores).

**La trampa del junior**: pensar que "hay una forma correcta" de autenticarse y tratar de usarla en todos los casos. Cada mecanismo sirve para un caso de uso distinto: algo desatendido (una función serverless corriendo sola) NO puede usar una sesión de usuario interactiva; algo que vos operás en vivo desde una terminal SÍ puede.

**Cómo lo piensa un senior**: elige el mecanismo según QUIÉN actúa en ese momento — ¿hay un humano presente pudiendo autenticarse interactivamente, o el código corre solo? Además, separa siempre secretos por su blast radius: una credencial que vive en un archivo versionado (git) es un incidente de seguridad esperando pasar; una credencial en un vault de secretos dedicado (no una tabla de tu propia base de datos) reduce la superficie de exposición.

**Tradeoffs reales**:

| Mecanismo | Sirve para | No sirve para | Dónde vive el secreto |
|---|---|---|---|
| API key | Procesos desatendidos (funciones serverless, cron jobs) | Nada — funciona siempre, pero es la opción menos segura si se filtra | Vault de secretos del runtime, NUNCA en una tabla propia ni en el repo |
| Sesión CLI / OAuth | Operación interactiva de un humano | Código corriendo sin supervisión | Token de sesión local, gestionado por la herramienta |
| MCP con OAuth | Que un LLM opere un servicio sin que el humano copie ninguna key a mano | Automatización 100% desatendida (igual necesita el flujo OAuth al menos una vez) | Depende del provider — a menudo no hay secreto en texto plano en ningún archivo |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué no usar siempre API key, si es la más simple?*
  A: Porque es la que más expone si se filtra — es un string estático que cualquiera con acceso puede usar. Para operación interactiva, sesión de usuario (revocable, atada a una persona) es más segura.
- Q (senior): *Tenés una función serverless que necesita hablar con un servicio externo. ¿Cómo decidís dónde guardar el secreto?*
  A: Nunca en una tabla de tu propia base (visible por cualquiera con acceso al panel de admin, y queda en backups en texto plano). Usar el mecanismo de secrets management nativo del runtime (vault de la plataforma), que expone el valor SOLO como variable de entorno dentro de la ejecución, nunca consultable por SQL.
- Q (trampa): *Una API key te da 401 y asumís que hay que regenerarla. ¿Qué chequeaste antes?*
  A: El formato exacto del header esperado (`Authorization: Bearer` vs `X-API-Key` no son intercambiables aunque ambos "suenen" a auth), el endpoint correcto, y recién después, si sigue fallando, la validez de la key en sí — con un test directo (curl) que aísle la key del resto del sistema.

## Lo que el libro NO tiene (gap externo — este hito es 100% gap)

Este hito no tiene capítulo en el libro de Imran Ahmad — es integración de sistemas, no diseño de agentes en sí. Fuentes primarias:

- **Webhooks, reintentos e idempotencia**: cada proveedor documenta su propia política — SIEMPRE leé la doc específica del proveedor que estés integrando antes de asumir garantías genéricas.
- **Anthropic tool use, `tool_choice` forzado**: https://docs.anthropic.com/en/docs/build-with-claude/tool-use — sección de `tool_choice` para forzar el uso de una tool específica cuando el flujo lo requiere.
- **OWASP LLM Top 10 — Excessive Agency**: https://genai.owasp.org/ — el riesgo de que un agente ejecute efectos con consecuencias sin validación suficiente está catalogado formalmente ahí.
- **Gestión de secretos serverless**: la doc específica de tu proveedor de funciones (Supabase Edge Functions Secrets, AWS Secrets Manager, Vercel Environment Variables, etc.) — el patrón es el mismo, el lugar exacto cambia por proveedor.

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `webhook-vs-polling`: armá un endpoint de webhook real (cualquier proveedor con sandbox gratis — WhatsApp, Slack, Stripe test mode) y mandate un evento de prueba. Mostrame el log de la request recibida.
- `agent-conversation-memory`: implementá un agente con tool-calling que arme su contexto desde el historial real de una conversación externa (no una tabla propia). Mostrame un caso donde el historial influye en la respuesta del turno actual.
- `tool-output-validation`: agregale a una tool con efectos (insert en base) una validación server-side contra un conjunto cerrado de valores. Mostrame el caso donde la validación rechaza un valor inválido sin romper el flujo.
- `external-platform-auth-patterns`: conectá un mismo servicio externo usando DOS mecanismos distintos (ej. API key para un caso desatendido, sesión CLI para uno interactivo) y explicame por qué cada uno corresponde a su caso.

### Para subir a `mastered`

- Integrá un agente end-to-end sobre una plataforma real de mensajería o webhook, con al menos: recepción por webhook, memoria desde el historial de la plataforma (curado, no crudo), al menos una tool con efecto validado server-side, y secretos gestionados en el vault del runtime (no en tu base). Feynman check: explicáselo a alguien que no sabe AI en 5 minutos, incluyendo por qué NINGUNA de las 4 piezas puede faltar sin agregar riesgo real.

## Anti-patterns que vas a ver en clientes reales

1. **Confiar en el texto de respuesta del agente como prueba de que algo se ejecutó**
   - Cómo se hace: el bot le dice al usuario "listo, guardado" y nadie chequea la base.
   - Por qué se hace: el mensaje "suena" a confirmación, se asume que implica éxito.
   - Costo real: datos que el usuario cree guardados y no existen — el peor tipo de bug, porque nadie se entera hasta que alguien busca algo que no está.
   - Cómo lo arregla un senior: la confirmación al usuario SOLO se manda después de que la tool confirma el efecto (ej. insert exitoso), nunca antes ni en paralelo.

2. **Secreto de proceso desatendido guardado en una tabla propia**
   - Cómo se hace: "no quiero usar la CLI de secretos, lo guardo en una tabla con RLS".
   - Por qué se hace: parece más simple, evita aprender el mecanismo nativo del proveedor.
   - Costo real: cualquiera con acceso al panel de admin de la base ve el secreto en texto plano; queda también en backups.
   - Cómo lo arregla un senior: usa el vault de secretos nativo del runtime (casi siempre configurable desde el dashboard web, sin CLI) — verificá antes de asumir que hay que elegir entre "conveniente" y "seguro".

3. **Pasarle al LLM su propio output pasado sin curar como memoria**
   - Cómo se hace: `history.push({role: "assistant", content: raw_platform_log})` sin revisar qué formato trae ese log.
   - Por qué se hace: es el camino de menor esfuerzo, "es solo texto".
   - Costo real: el modelo empieza a imitar artefactos de formato de logging como si fueran su propio estilo de respuesta — errores sutiles, difíciles de diagnosticar sin mirar el historial exacto que se le pasó.
   - Cómo lo arregla un senior: normaliza o resume cada entrada del historial antes de pasarla como contexto — pregúntate "¿esto es lo que YO quiero que el modelo aprenda a decir?".

4. **Máquina de estados propia cuando la plataforma ya trackea el estado**
   - Cómo se hace: armar una tabla `conversaciones` con columna de "paso actual" sin chequear si el proveedor ya expone esa información.
   - Por qué se hace: es el patrón más familiar (viene de desarrollo web tradicional).
   - Costo real: dos fuentes de verdad que se pueden desincronizar, más código para mantener.
   - Cómo lo arregla un senior: primero pregunta si el historial/estado de la plataforma alcanza. Si alcanza, usalo. Solo agregá estado propio cuando el historial genuinamente no cubre lo que necesitás saber.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés explicar la diferencia entre webhook y polling, y qué implica idempotencia para tu endpoint?
- [ ] ¿Podés diseñar la memoria de un agente usando el historial de una plataforma externa, curándolo para que el modelo no imite artefactos de formato?
- [ ] Si te dan una tool con efectos reales (insertar, cobrar, enviar), ¿podés listar qué validás server-side ANTES de ejecutar, más allá de lo que dice el system prompt?
- [ ] ¿Podés elegir el mecanismo de autenticación correcto (API key / sesión / MCP) según si hay un humano presente o el proceso corre desatendido?
- [ ] ¿Podés explicar por qué el texto de respuesta de un agente NUNCA es prueba suficiente de que un efecto se ejecutó?
