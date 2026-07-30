# Chuleta — Hito 1: Fundamentos

> Referencia rápida. Para aprender de cero, andá a `milestones/01-fundamentals.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `react-loop` | Loop Thought→Action→Observation hasta Final Answer | "Es una state machine implícita cuyo history crece O(n²); ReAct es UNA familia de loops, no EL algoritmo." |
| `json-mode` | Forzar JSON válido por constrained decoding | "JSON mode no es prompting, es una constraint a nivel de sampler — la diferencia entre rezar y tener un contrato." |
| `function-calling` | El LLM PIDE ejecutar una función; vos la ejecutás | "El modelo nunca toca tu código. VOS sos el sandbox; cada arg es input untrusted." |
| `memory-tiers` | Working / episodic / semantic | "Cada tier tiene storage, refresh y latency budget distintos. Meter logs en vector store es pagar 4x por lo que un SELECT resuelve." |
| `prompt-patterns` | PTCF, CoT, ToT, Few-Shot, Self-Consistency | "Cada pattern resuelve un problema distinto. Elegís midiendo el tipo de error, no mezclando todo." |

## Tradeoff principal del hito — loops de agente

| Loop | Conviene | NO conviene |
|---|---|---|
| ReAct puro | <10 pasos, observations chicos, exploratorio | pasos largos, multi-tenant SLA estricto |
| Plan-and-Execute | flow conocido upfront | el plan cambia según resultados |
| Reflexion | calidad > latencia | real-time chat |
| LangGraph DAG | producción, checkpoints/HITL | MVP rápido, demo |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Agente en loop infinito sin `max_iterations` → max_iterations explícito + early-stop por repetición + fallback a Final Answer con el error.
2. Parsear JSON con regex → Structured Outputs (OpenAI) o tool use schema (Anthropic). Fin.
3. Mandar todo el history sin compactar → sliding window + summarization periódica + episodic memory para queries específicas.

## La pregunta de entrevista que más cae

**Q:** Tu agente entra en loop infinito haciendo la misma tool call. ¿Qué hacés?
**A (esqueleto):**
- Tres causas: tool devuelve error que el LLM no lee como "stop"; prompt sin criterio de stop; falta hard limit.
- Fix: max_iterations + early-stopping por repetición de actions + mejorar prompt.
- El delta senior: agregar fallback a Final Answer con explicación humana de la falla (no dejar al user sin respuesta).

## Decisión rápida (cheat)

- **¿JSON mode o pedir JSON en el prompt?** Si actuás sobre el output (lo parseás, lo ejecutás) → Structured Outputs/tool schema siempre. "Pedir JSON" solo para prototipo descartable.
- **¿Dónde guardo esta info?** ¿Cambia por turno? → working. ¿Es un evento con timestamp? → episodic (Postgres/Redis). ¿Es verdad atemporal estable? → semantic (vector/KG). Nunca todo en el prompt.
- **¿CoT o zero-shot?** Lógica/math/multi-step → CoT. Classification simple → zero-shot (CoT es overhead sin beneficio).
- **¿Cuántos few-shot?** Empezá con 3-5 curados. Más NO es mejor: sube costo y a partir de cierto N el beneficio se vuelve negativo.
