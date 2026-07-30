# Hito 3 — APIs & Microservicios

## Por qué importa (perspectiva corporativa)

Acá está el tema, y prestá atención porque esto es lo que separa al "AI Engineer que arma demos" del "AI Engineer que pone agentes en producción": un agente LLM en producción NO es un script de Python que llama a OpenAI. Es un **sistema distribuido** con latencia variable (5s a 60s típico), llamadas costosas (USD 0.01 a USD 1 por request), rate limits asimétricos por provider, y tokens que se streamean de a uno. Si no manejás async, SSE, rate limits, caching y cost optimization, tu producto **NO escala** y se vuelve **insostenible económicamente** en 3 meses.

Esto es lo que se ve TODO el tiempo en clientes: startup arma agente en notebook, funciona divino para 5 usuarios, lo deployan en un Flask sync, llega a 50 usuarios concurrentes y el server se cae porque cada request bloquea 30 segundos esperando a OpenAI. O peor: pagan USD 8000/mes en API porque NUNCA implementaron prompt caching y mandan el mismo system prompt de 4k tokens en cada uno de los 100k requests diarios. Un senior con este hito sólido baja ese costo a USD 800 — un orden de magnitud, sin tocar la calidad.

Las oportunidades laborales: "AI Platform Engineer" en empresas con >1M requests/mes a LLMs (cualquier scaleup con producto AI), "Cost Optimization Consultant" freelance (esto es ORO ahora — empresas pagan 100-200 USD/hora porque cualquier mejora de 20% en costos de LLM les paga el consultor 10x), "Backend Engineer" especializado en LLM systems (Anthropic Solutions, OpenAI partner engineering). En 2026 no podés ser AI Engineer Senior sin saber esto. Punto.

## Conceptos de este hito

### async-patterns

**Qué es**: Patrones de concurrencia con `async/await` (Python asyncio, JS Promise) para hacer N llamadas a LLM en paralelo sin bloquear, con control de concurrencia (semáforos), backpressure, y manejo de errores parciales.

**La trampa del junior**: Llamar al LLM con código sync (`requests.post(...)`) dentro de un endpoint web. O al revés: meter `asyncio.gather()` con 500 calls en paralelo y matar la rate limit del provider, recibir 429 en cascada, y romper todo el sistema.

**Cómo lo piensa un senior**: Async NO es "más rápido". Async es **mejor uso de I/O wait**. Mientras esperás respuesta de OpenAI (30s), el event loop puede atender otros 100 requests. Sin async, cada request bloquea un thread/worker entero. La gestión correcta requiere: (1) cliente async (`AsyncOpenAI`, `AsyncAnthropic`, `httpx.AsyncClient`), (2) **semáforo** para limitar concurrencia (no más de N llamadas simultáneas — protege rate limit y tu propia memoria), (3) `gather` con `return_exceptions=True` para no perder todo si UNA falla, (4) timeout explícito por llamada.

**Tradeoffs reales**:

| Approach | Cuándo |
|---|---|
| Sync (`requests`) | Script offline one-shot, prototipo |
| Async + sin límite (`gather(*tasks)`) | NUNCA en producción — mata rate limit |
| Async + Semaphore | Default en backend web |
| Async + queue (Celery, RQ) | Cargas pesadas, retries con persistencia, jobs largos |
| asyncio + httpx + tenacity | Stack moderno completo |
| Sync + threads (concurrent.futures) | Lenguajes/frameworks sin async maduro |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué async ayuda con llamadas LLM?*
  A: Porque las llamadas LLM son I/O bound — pasás 99% del tiempo esperando respuesta. Async libera el event loop durante esa espera para procesar otros requests. Sin async, un único request bloquea el worker entero durante 10-30s, matando throughput.
- Q (senior): *¿Cómo controlás concurrencia para no romper rate limits?*
  A: `asyncio.Semaphore(N)` donde N se calcula según el RPM permitido del provider y la latencia media de la llamada. Ej: si Anthropic te da 1000 RPM y cada call tarda 10s, podés tener ~166 concurrent calls sin pasarte. Combiná con tenacity para retries con exponential backoff sobre 429.
- Q (trampa, system design): *Tu agente hace 5 tool calls en paralelo. Una falla con 500. ¿Qué hacés?*
  A: Depende de la semántica. Si las 5 son independientes y "todo o nada" no es requerido: `gather(*tasks, return_exceptions=True)` + procesás los exitosos + reintentás el fallido + le devolvés al LLM el partial result con nota del error. Si son dependientes: cancelás todo, retry con backoff, y si sigue fallando, devolvés error contextualizado al LLM para que decida fallback. La trampa: muchos hacen `gather` sin `return_exceptions` y pierden TODO si una falla.

### sse-streaming

**Qué es**: **Server-Sent Events** — protocolo HTTP de streaming unidireccional server→client. El estándar de facto para streamear tokens de LLM al frontend en tiempo real. El usuario ve la respuesta apareciendo letra por letra en vez de esperar 30s en blanco.

**La trampa del junior**: Implementar streaming pero no testearlo bajo reverse proxy. Nginx por default **buffea** la respuesta y rompe SSE. El user sigue viendo el blanco hasta que termina toda la respuesta. Encima difícil de debuggear porque "en local funcionaba".

**Cómo lo piensa un senior**: SSE no es solo "mostrar tokens en tiempo real" — es **arquitectura completa** con implicancias. Tenés que manejar: (1) **chunking** del response del provider y forwarding al cliente, (2) **errores parciales** (¿qué hacés si el stream se corta a medio token?), (3) **cancelación** (el user cierra la pestaña, ¿matás la llamada al provider para no pagar?), (4) **buffering del proxy** (`X-Accel-Buffering: no` header), (5) **tool calls en streaming** (function calls se streamean en deltas, hay que reconstruirlos), (6) **costo per stream** (cobrás por tokens generados aunque el user canceló).

**Tradeoffs reales**:

| Approach | Pro | Contra |
|---|---|---|
| SSE (HTTP) | Simple, traversa proxies, reconnect built-in | Unidireccional, request-response |
| WebSockets | Bidireccional, baja latencia | Más infra, problemas con CDN/proxies, statefull |
| Polling | Funciona en cualquier infra | Latencia alta, costo en server |
| Streaming completo (sin chunking) | Más simple | Usuario espera 30s en blanco — UX horrible para chat |
| gRPC streaming | Performance interna entre services | No directo al browser sin gateway |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es SSE y por qué se usa para streaming de LLMs?*
  A: Server-Sent Events. Protocolo HTTP donde el server mantiene conexión abierta y manda eventos `data: ...\n\n` al cliente. Se usa para LLMs porque (a) los tokens llegan progresivos del provider, (b) el user no debe esperar 30s en blanco, (c) browsers lo soportan nativo via `EventSource`, (d) atraviesa proxies y CDNs sin drama (mejor que WebSocket).
- Q (senior): *Tu stream SSE funciona local pero en prod los tokens llegan todos juntos al final. ¿Por qué?*
  A: Casi seguro buffering en algún hop. Causas típicas: (1) Nginx con `proxy_buffering on` (default) — fix: `proxy_buffering off` para esa location o header `X-Accel-Buffering: no` en response, (2) Cloudflare/CDN que buffea — desactivar para esa ruta, (3) algún middleware web (FastAPI con `gzip` middleware buffea el chunk). El test: `curl --no-buffer` directo al server y comparar con curl al endpoint público.
- Q (trampa): *Usuario cancela request en streaming. ¿Qué hacés del lado server?*
  A: La pregunta esconde 3 capas. (1) Detectar el cancel: en FastAPI/Starlette `request.is_disconnected()` o en raw async leer el client disconnect. (2) Cancelar tu llamada al provider — IMPORTANTE: si NO cancelás, el provider sigue generando y vos seguís PAGANDO por tokens que nadie ve. Usá `asyncio.CancelledError` propagation o el cancel token del SDK. (3) Loggear como "cancelled" para métricas, no como error. La trampa: mucha gente solo cierra la conexión al cliente y deja el upstream generando.

### rate-limits

**Qué es**: Límites del provider sobre **RPM** (requests per minute), **TPM** (tokens per minute), y a veces **RPD** (requests per day) o concurrent requests. Cada provider tiene su propio régimen y tier system.

**La trampa del junior**: Asumir que el límite que ve en la docs (ej "60 RPM en tier 1") es estático. NO. Los límites varían por modelo, por org, por método de pago, y por uso histórico. Encima los errores 429 vienen con `Retry-After` header que casi nadie respeta.

**Cómo lo piensa un senior**: Rate limits son **constraint de arquitectura**, no excepción a manejar. Senior diseña con: (1) **client-side rate limiter** que respeta el TPM/RPM del provider (ej `aiolimiter` o token bucket custom), (2) **exponential backoff con jitter** en 429, (3) **fallback a otro provider o modelo** (degradación grácil — Claude 3.5 → GPT-4o → Haiku), (4) **distribución entre múltiples API keys** si la org lo permite, (5) **monitoring**: tasa de 429, latencia P95, costo por hora — todo en Grafana/Datadog.

**Tradeoffs reales**:

| Estrategia | Cuándo |
|---|---|
| Retry con backoff (tenacity) | Picos esporádicos, default razonable |
| Client-side rate limiter | Workloads constantes y altos, evitar 429 |
| Multi-key distribution | Volumen muy alto, contrato lo permite |
| Multi-provider fallback | Resiliencia + cost optimization combinados |
| Queue + worker pool (Celery/SQS) | Workloads batch, jobs largos |
| Increase tier (paying more upfront) | Cuando el costo del workaround supera al upgrade |

**En entrevista te van a preguntar**:
- Q (mid): *Diferencia entre RPM y TPM.*
  A: RPM = requests por minuto (cuántas llamadas hacés). TPM = tokens por minuto (cuánto volumen de tokens — input + output — procesás). Una llamada con 50k tokens cuenta 50k contra TPM aunque sea solo 1 RPM. Ambos se cuentan, y el que llegue antes a su límite te tira 429.
- Q (senior): *¿Cómo manejás 429 en producción?*
  A: Stack típico: (1) `tenacity` con `wait_exponential_jitter` y `retry_if_exception_type(RateLimitError)`, (2) respetar `Retry-After` header si viene, (3) max retries 3-5, después escalar a fallback (otro modelo o cola de retry persistente), (4) alertar si la tasa de 429 supera 1% (señal de capacity planning), (5) registrar cada 429 con request_id para postmortem. NUNCA infinite retry.
- Q (trampa, system design): *Tu app necesita 200 RPM sostenidos pero tu tier permite 150 RPM. ¿Qué hacés?*
  A: Trampa típica — "subo de tier". A veces es lo correcto, a veces no. Análisis: (1) ¿Es spike puntual o sostenido? Si spike, queue con worker pool resuelve. (2) ¿Se puede hacer prompt caching para reducir llamadas efectivas? (3) ¿Hay calls duplicadas (cache de respuestas a queries idénticas)? (4) ¿Multi-key con segunda org es viable y legal en el contrato? (5) ¿Multi-provider fallback (Claude + OpenAI alternados) saca presión? Subir de tier es opción 5, no opción 1. La trampa: tirar plata sin optimizar.

### prompt-caching

**Qué es**: Cachear el **prefix estable** del prompt (system prompt + few-shots + context grande) en el lado del provider para que requests siguientes paguen solo los nuevos tokens. Anthropic, OpenAI y Google lo soportan con APIs distintas.

**La trampa del junior**: Pensar que prompt caching es "como Redis pero del prompt". NO ES ESO. Es caching EN EL PROVIDER que reduce **costo** (Anthropic 90% off en hits, OpenAI 50% off) y **latencia** (~85% más rápido el TTFB). No es algo que vos implementás client-side — vos solo estructurás el prompt correctamente y marcás los cache breakpoints.

**Cómo lo piensa un senior**: Prompt caching es **el optimizador #1 de costo en sistemas con system prompts grandes o RAG**. Pero requiere disciplina arquitectónica: el **prefix debe ser idéntico byte-a-byte** entre requests. Cualquier variación (timestamp en el system prompt, nombre del user, fecha actual) invalida el cache. Senior estructura el prompt en bloques: `[cached: system_prompt + tools + few-shots + RAG_context] + [not_cached: user_query + dynamic_vars]`. TTL del cache: Anthropic 5min (ephemeral) o 1h (extended), OpenAI ~5-10min auto.

**Tradeoffs reales**:

| Provider | Cómo funciona | Discount | TTL | Mínimo |
|---|---|---|---|---|
| Anthropic | Explícito (`cache_control: ephemeral`) hasta 4 breakpoints | 90% read, +25% write | 5min default, 1h opcional | 1024 tokens (Sonnet) |
| OpenAI | Automático, prefijo de >1024 tokens | 50% read (input cached) | 5-10min auto | 1024 tokens |
| Google Gemini | Context caching API explícita | Hasta 75% | Configurable | 32k tokens (Pro) |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es prompt caching y qué ahorrás?*
  A: Caching del prefix estable del prompt en el lado del provider. Beneficios: hasta 90% reducción de costo en input tokens cacheados (Anthropic) y 85% reducción de TTFB. Crítico para systems con system prompts largos o RAG con docs repetidos entre queries.
- Q (senior): *¿Cómo estructurás un prompt para maximizar cache hit rate?*
  A: Layout obligatorio: prefix ESTABLE primero (system + tools + few-shots + RAG context con docs largos), variables DINÁMICAS al final (user query, timestamp, vars que cambian). En Anthropic: cache_control en el último bloque estable. CERO variables interpoladas en el prefix — un timestamp en el system prompt te tira cache hit rate a 0%.
- Q (trampa): *Tu cache hit rate es 0% aunque el system prompt parece idéntico. ¿Qué buscás?*
  A: Causas típicas: (1) variable interpolada que cambia (fecha, user_id, request_id) en el bloque "estable", (2) ordering inconsistente de tools en la lista, (3) whitespace diferente (un \n extra), (4) cache_control puesto antes de que el prefix sume el mínimo de tokens (1024 Sonnet, distinto Haiku), (5) TTL vencido (>5min entre requests). Tool de diagnóstico: dumpear el prompt byte a byte y diff entre 2 requests "iguales".

### cost-optimization

**Qué es**: Conjunto de prácticas para reducir costo total de LLM en producción manteniendo calidad: **model routing** (modelo chico para tareas fáciles, grande para difíciles), **batch API** (50% off, async), **prompt caching**, **response caching** (Redis client-side de queries comunes), **streaming cancel** (no pagar lo que el user no ve), **token budgets**, **eval-driven model selection**.

**La trampa del junior**: Usar gpt-4o o claude-opus para TODO. "Es el mejor modelo, total". Costo 30x el necesario, latencia 3x. O al revés: usar siempre el modelo más barato y entregar calidad de mierda.

**Cómo lo piensa un senior**: Cost optimization es **discipline ongoing**, no proyecto. Sistema en producción serio tiene: (1) **dashboard de costo por feature/tenant** (cost-attribution), (2) **model router**: clasificador chico decide qué modelo usar según task, (3) **batch API** para todo lo no-realtime (50% off Anthropic + OpenAI), (4) **prompt caching agresivo**, (5) **response cache** (Redis) para queries idénticas/similares (hash de query + version del system prompt), (6) **token budgets** por usuario/feature (cortar en exceso), (7) **A/B test continuo** modelo barato vs caro — si la métrica de calidad no baja, te quedás con el barato. Empresas que NO hacen esto pagan 5-10x lo necesario.

**Tradeoffs reales**:

| Técnica | Saving típico | Trade-off |
|---|---|---|
| Model routing (chico→grande) | 60-80% | Necesita router robusto, sino calidad cae |
| Batch API (async, ≤24h) | 50% | No realtime — solo offline jobs |
| Prompt caching | 60-90% (sobre prefix) | Requiere disciplina del prompt layout |
| Response caching | 100% en hits | Solo queries idempotentes, gestionar invalidación |
| Streaming cancel | Variable | Implementación cuidadosa de cancel propagation |
| Modelo open-source self-hosted | 50-95% según volumen | Infra GPU, ops, calidad variable según modelo |
| Distillation (fine-tune chico con outputs grande) | 80-95% | Inversión inicial alta, mantenimiento |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué usar batch API si tarda hasta 24h?*
  A: 50% de descuento en input + output. Sirve para todo lo NO-realtime: indexado/embedding de corpus, evaluaciones, summarización offline, classification de backlogs, generación de datasets sintéticos. Cualquier cosa donde la respuesta no la espera un humano en chat.
- Q (senior): *Tu empresa paga USD 50k/mes en OpenAI. ¿Por dónde atacás?*
  A: Orden típico de impact: (1) auditá si hay prompt caching mal implementado o ausente (suele dar 30-50% sólo eso si el system prompt es grande). (2) Identificá feature top de costo (Pareto: 1-2 features suelen ser 60-80% del bill). (3) Model routing: ¿hay tasks fáciles usando gpt-4o que podrían ir a gpt-4o-mini? (4) Cache de respuestas para queries repetitivas (FAQ-like). (5) Migrar batches a Batch API. (6) Evaluar Claude Haiku o Gemini Flash como segundo provider para fallback/diversidad de costo. Cada paso medido con eval — no hagas saving que rompa calidad.
- Q (trampa, system design): *Te proponen migrar todo a un modelo open-source self-hosted (Llama 3.3 70B). ¿Aceptás?*
  A: Depende. Análisis: (1) Costo total = GPU rental (L40/H100) + ops (engineers) + opportunity cost de no tener última feature del provider. Solo es break-even típicamente >5k requests/h sostenidos. (2) Calidad: Llama 3.3 está cerca pero NO igual a GPT-4o o Sonnet 3.5. Hay que medir en TU dominio. (3) Latencia: self-host te da control fino pero requiere optimización (vLLM, TensorRT-LLM). (4) Compliance: a veces self-host es REQUERIDO (datos sensibles, residencia). La trampa: decir "sí siempre" o "no siempre". Decisión empresarial, no técnica de un lado solo.

## Lo que el libro hace bien acá

- **chapter04** — `Agent Deployment & Responsible Development` — toca cost-aware routing, circuit breakers, infrastructure scaling. Decente para entender la mentalidad de deployment, aunque no entra a profundidad en async/streaming. El section de cost-aware routing es buen warm-up.

## Lo que el libro NO tiene (gaps a saber)

Este hito es **el más débil del libro**. Casi todo es gap externo. Si solo estudiás del libro y querés competir como AI Engineer Senior, te falta esta capa completa. Andá a estos recursos sí o sí:

- **Async patterns con asyncio**: el libro hace casi todo sync.
  - Recurso: https://docs.python.org/3/library/asyncio.html + `AsyncOpenAI` / `AsyncAnthropic` docs.
  - Qué entender: event loop, `gather` con `return_exceptions`, `Semaphore` para concurrency control, `timeout` por call, `aiolimiter` para rate limiting client-side.

- **SSE streaming en backend Python (FastAPI)**: ausente del libro.
  - Recurso: https://platform.openai.com/docs/api-reference/streaming + https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse + https://docs.anthropic.com/en/api/messages-streaming
  - Qué entender: cómo cada provider streamea (deltas), reconstrucción de tool calls desde deltas, manejo de cancelation (`request.is_disconnected()`), headers anti-buffering para Nginx (`X-Accel-Buffering: no`).

- **Rate limits específicos por provider**: el libro generaliza.
  - Recurso: https://docs.anthropic.com/en/api/rate-limits + https://platform.openai.com/docs/guides/rate-limits + https://ai.google.dev/gemini-api/docs/rate-limits
  - Qué entender: tier system de cada uno, cómo escalan los límites con uso histórico, métodos de pago que aceleran upgrade, `Retry-After` header.

- **Prompt caching (Anthropic + OpenAI + Gemini)**: gap CRÍTICO del libro.
  - Recurso: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching + https://platform.openai.com/docs/guides/prompt-caching + https://ai.google.dev/gemini-api/docs/caching
  - Qué entender: cuándo se cachea automático vs manual, breakpoints en Anthropic, TTL (5min ephemeral, 1h extended), el cache_creation_input_tokens vs cache_read_input_tokens en la billing response.

- **Batch APIs**: cero del libro.
  - Recurso: https://platform.openai.com/docs/guides/batch + https://docs.anthropic.com/en/api/messages-batches
  - Qué entender: cuándo usar batch (offline, hasta 24h SLA), 50% discount, cómo armar un batch file JSONL, polling de status, manejo de errors per-request.

- **Cost optimization patterns**: el libro lo toca superficialmente en ch4.
  - Recurso: https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching#use-cases + posts del blog de Anthropic sobre cost reduction case studies.
  - Qué entender: model router con classifier chico (`gpt-4o-mini` o un BERT chico) decidiendo modelo, response cache con Redis, token budgets, A/B test continuo.

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `async-patterns`: NO hay notebook directo. Tomá CUALQUIER agente del libro (ej chapter01) y migralo de sync a async usando `AsyncOpenAI` o `AsyncAnthropic`. Implementá un loop que haga 20 queries en paralelo con `Semaphore(5)`. Pegame código + tiempo total comparado con la versión sync.
- `sse-streaming`: levantá un FastAPI con UN endpoint que streamee desde OpenAI/Anthropic. Probalo con `curl --no-buffer`. Pegame código del endpoint + ejemplo de la salida streamed.
- `rate-limits`: implementá un retry con `tenacity` que respete `Retry-After` y haga exponential backoff con jitter sobre 429. Forzá rate limit (loop de 200 calls rápido) y mostrame que la app sobrevive.
- `prompt-caching`: con Anthropic, armá un script que llame 5 veces a Claude con un system prompt de >2k tokens. Primero sin `cache_control`, después con. Mostrame los `cache_creation_input_tokens` y `cache_read_input_tokens` de cada call.
- `cost-optimization`: tomá un agente del libro que use gpt-4o. Migralo a gpt-4o-mini para una sub-tarea (ej classification) manteniendo gpt-4o para la generación final. Medí costo total por run antes/después.

### Para subir a `mastered`

- `async-patterns`: en un proyecto propio, implementá un endpoint que orquesta 5 tool calls en paralelo con semaphore + retry + timeout + partial failure handling. Defendé el N del semaphore con cálculo de RPM permitido.
- `sse-streaming`: deployá streaming end-to-end (browser frontend `EventSource` → tu FastAPI → provider) en un entorno real con Nginx delante. Resolvé el buffering. Implementá cancel propagation (cancelás la llamada al provider cuando el user cierra). Feynman check: explicáme por qué SSE > WebSocket para chat con LLMs.
- `rate-limits`: documentá el rate-limit budget de tu app (RPM, TPM esperados, tier actual, headroom). Diseñá la estrategia de degradación grácil cuando se acerca al límite (queue, fallback provider, etc).
- `prompt-caching`: en proyecto real con system prompt grande, medí cache hit rate antes y después de optimizar el prompt layout. Mostrame el ahorro en USD/mes proyectado.
- `cost-optimization`: en un proyecto real con bill >USD 500/mes, ejecutá un cost audit (Pareto por feature) y proponé 3 optimizaciones priorizadas por ROI. Implementá la #1, medí impact. Feynman check: explicáme cómo decidirías si self-hostear Llama 70B vale o no para X workload.

## Anti-patterns que vas a ver en clientes reales

1. **Endpoint sync llamando a OpenAI**
   - Cómo se hace: Flask + `requests.post(openai_url, ...)` directo en el handler.
   - Por qué se hace: "es lo que sale del tutorial".
   - Costo real: 1 worker bloqueado 20s por request = ~3 RPS de throughput por worker. Para 100 concurrent users necesitás 30+ workers (RAM). App se cae con 50 users.
   - Cómo lo arregla un senior: migrar a FastAPI + async + AsyncOpenAI/AsyncAnthropic. Throughput sube 10-50x sin más infra.

2. **`asyncio.gather` sin Semaphore sobre 500 calls**
   - Cómo se hace: `await gather(*[call_llm(x) for x in big_list])`.
   - Por qué se hace: "async = paralelo = mejor".
   - Costo real: rate limit 429 en cascada, retries exponenciales que empeoran el bloom, provider banea temporalmente la org.
   - Cómo lo arregla un senior: `Semaphore(N)` calculado según RPM/TPM, o queue + worker pool. Concurrency control es OBLIGATORIO.

3. **Streaming sin cancelar la llamada upstream cuando el user cancela**
   - Cómo se hace: response stream con `async for token in llm.stream(...)`, sin chequear si el cliente sigue conectado.
   - Por qué se hace: "el browser cerró, total".
   - Costo real: pagás por tokens generados que NADIE ve. En productos con muchos usuarios cancelando (ej chat), 10-20% del bill puede ser tokens fantasma.
   - Cómo lo arregla un senior: chequear `request.is_disconnected()` en el loop de stream + cancelar la llamada al provider via task cancel.

4. **System prompt de 4k tokens enviado en cada request sin caching**
   - Cómo se hace: prompt template con `system="..."` largo, mandado fresh cada vez.
   - Por qué se hace: no conocen prompt caching o "después lo implemento".
   - Costo real: 100k requests/día × 4k tokens input × $3/1M (Sonnet) = USD 1200/día solo en system prompt. Con caching baja a ~USD 120/día.
   - Cómo lo arregla un senior: cache_control en el system prompt + medir cache_read_input_tokens. Es 10 líneas de código que devuelven 10x ROI.

5. **Mismo modelo (el más caro) para todo**
   - Cómo se hace: `model="claude-3-opus"` o `model="gpt-4o"` en cada call.
   - Por qué se hace: "no sé qué modelo elegir para qué, así que el mejor".
   - Costo real: 10-30x sobre lo necesario. Tareas como "clasificá este ticket en una de 5 categorías" se resuelven con haiku/4o-mini a costo despreciable.
   - Cómo lo arregla un senior: model router (un clasificador chico, o un prompt simple a un modelo barato) que decide qué modelo invocar según task. A/B test continuo para validar que la calidad no cae.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés escribir un endpoint async que llama a LLM con Semaphore + timeout + retry + cancel propagation?
- [ ] ¿Podés implementar SSE streaming end-to-end (provider → server → browser) y debuggear buffering issues en Nginx?
- [ ] ¿Conocés el régimen de rate limits de Anthropic, OpenAI y Google, y podés diseñar estrategia de fallback entre ellos?
- [ ] ¿Podés explicar exactamente cómo funciona prompt caching en Anthropic vs OpenAI, calcular ahorro proyectado, y debuggear cache_hit_rate=0%?
- [ ] ¿Podés hacer cost audit Pareto de un sistema LLM real, priorizar 3 optimizaciones, e implementarlas midiendo impact?
- [ ] En entrevista senior, ¿podés defender model routing vs "siempre el mejor modelo" con tradeoffs cuantitativos?
