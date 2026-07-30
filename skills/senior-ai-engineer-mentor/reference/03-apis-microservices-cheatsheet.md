# Chuleta — Hito 3: APIs & Microservicios

> Referencia rápida. Para aprender de cero, andá a `milestones/03-apis-microservices.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `async-patterns` | async/await + Semaphore para N calls en paralelo | "Async no es 'más rápido', es mejor uso del I/O wait. Semaphore es OBLIGATORIO en prod." |
| `sse-streaming` | Server-Sent Events: tokens al frontend en vivo | "Funciona local, falla en prod por buffering de Nginx (`X-Accel-Buffering: no`)." |
| `rate-limits` | RPM/TPM del provider, 429 | "Rate limits son constraint de arquitectura, no excepción a manejar." |
| `prompt-caching` | Cachear el prefix estable en el PROVIDER | "Una variable interpolada en el prefix te tira el cache hit rate a 0%." |
| `cost-optimization` | Routing, batch, caching, cancel, budgets | "Es disciplina ongoing, no proyecto. Subir de tier es opción 5, no opción 1." |

## Tradeoff principal del hito — técnicas de cost optimization

| Técnica | Saving típico | Trade-off |
|---|---|---|
| Model routing (chico→grande) | 60-80% | necesita router robusto |
| Batch API (≤24h) | 50% | solo offline, no realtime |
| Prompt caching | 60-90% sobre prefix | disciplina del layout |
| Response caching | 100% en hits | solo queries idempotentes |
| Streaming cancel | variable | cancel propagation cuidadoso |
| Self-host open-source | 50-95% | infra GPU + ops |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Endpoint sync (`requests.post`) llamando a OpenAI → FastAPI + async + AsyncOpenAI/AsyncAnthropic; throughput sube 10-50x sin más infra.
2. `asyncio.gather` sin Semaphore sobre 500 calls → `Semaphore(N)` según RPM/TPM, o queue + worker pool. Concurrency control obligatorio.
3. System prompt de 4k tokens sin caching → `cache_control` en el prefix; 10 líneas de código, 10x ROI.

## La pregunta de entrevista que más cae

**Q:** Tu empresa paga USD 50k/mes en LLMs. ¿Por dónde atacás?
**A (esqueleto):**
- Auditá prompt caching ausente/mal hecho (30-50% solo eso si el system prompt es grande).
- Pareto por feature: 1-2 features suelen ser 60-80% del bill.
- Model routing: tasks fáciles en gpt-4o que podrían ir a mini.
- Response cache para queries repetitivas.
- Migrar batches a Batch API.
- Cada paso medido con eval — no rompas calidad por ahorrar.

## Decisión rápida (cheat)

- **¿Subo de tier?** Primero: ¿es spike (→ queue) o sostenido? ¿hay prompt caching? ¿hay calls duplicadas (→ response cache)? ¿multi-key/multi-provider viable? Subir de tier es lo ÚLTIMO.
- **¿Cómo calculo el N del Semaphore?** N ≈ RPM_permitido × (latencia_media_seg / 60). Ej: 1000 RPM, calls de 10s → ~166 concurrent.
- **¿SSE o WebSocket?** Para chat con LLM → SSE (unidireccional, atraviesa proxies/CDN, reconnect nativo). WebSocket solo si necesitás bidireccional real.
- **¿Por qué cache hit rate 0%?** Buscá variable interpolada en el prefix (fecha, user_id, request_id), ordering inconsistente de tools, whitespace, prefix bajo el mínimo (1024 tokens), o TTL vencido (>5min).
- **Tokens fantasma:** si el user cancela y NO cancelás upstream, pagás por tokens que nadie ve (10-20% del bill en chat).
