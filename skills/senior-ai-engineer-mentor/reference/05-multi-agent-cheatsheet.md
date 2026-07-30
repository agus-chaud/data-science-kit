# Chuleta — Hito 5: Multi-Agent

> Referencia rápida. Para aprender de cero, andá a `milestones/05-multi-agent.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `supervisor-pattern` | Jefe rutea a workers especializados | "El default razonable. Workers con descripciones MUTUAMENTE EXCLUYENTES o el routing es inestable." |
| `hierarchical-pattern` | Supervisores anidados (N niveles) | "Para dominios DISJUNTOS. 4+ niveles = problema de modelado, no de jerarquía." |
| `horizontal-network` | Agentes peer-to-peer sin jefe | "En apps de negocio casi siempre es mala elección: deadlocks, debug miserable." |
| `task-delegation` | A quién delega y con QUÉ contexto | "La segunda parte (handoff prompting) es la que casi nadie piensa: no pasar la conversación entera." |
| `conflict-resolution` | Resolver acciones incompatibles | "Es parte del DISEÑO, no afterthought. Siempre un fallback que SIEMPRE resuelve." |

## Tradeoff principal del hito — topologías multi-agent

| Pattern | Pro | Contra |
|---|---|---|
| Supervisor (centralizado) | control claro, fácil debug | single point of failure, bottleneck |
| Hierarchical | escala, ownership por dominio | múltiples hops, latency |
| Horizontal network | flexibilidad, resilience, simulaciones | deadlocks, hard to debug, costo impredecible |
| Single-agent + tools | menos latencia/costo/complejidad | no para sub-tasks genuinamente heterogéneas |

## Top 3 anti-patterns (con el fix en 1 línea)

1. "Vamos a hacerlo multi-agent" sin justificar → empezá single-agent-with-tools; migrá solo ante fricción CONCRETA (no hipotética).
2. Sin `max_iterations`/`recursion_limit` en loops multi-agent → límite duro + early-stopping por repetición + fallback a "escalo a humano".
3. Conflictos "el último que escribe gana" (state sin reducer) → reducers explícitos + mecanismo de conflict resolution por campo.

## La pregunta de entrevista que más cae

**Q:** Defendé single-agent-with-tools contra un interrogador que insiste en multi-agent.
**A (esqueleto):**
- El 80% de los casos "multi-agent" los resuelve mejor UN agente bien diseñado con tools.
- Multi-agent agrega latencia (3-5x), costo (2-10x), debugging miserable y modos de falla nuevos (deadlocks, infinite delegation).
- El 20% legítimo: sub-tasks heterogéneas (research+writing+fact-check), especialización profunda, simulaciones.
- Migrar solo ante fricción concreta medida, documentando la decisión.

## Decisión rápida (cheat)

- **¿Multi-agent o single-agent?** ¿Las sub-tasks son genuinamente heterogéneas y necesitan especialización profunda? → multi-agent. Si no → single-agent + tools (más barato, más rápido, más fácil de debuggear).
- **¿Supervisor flat o hierarchical?** Flat por default. Hierarchical solo con ≥3 dominios disjuntos claros, cada uno con ≥3 workers, donde un supervisor único sería intratable.
- **¿LLM-as-supervisor o classifier para routing?** Bajo volumen / casos variados → LLM. Alto volumen (>10 RPS) / casos estables → classifier (50ms, 100x más barato). Híbrido: classifier + LLM fallback bajo threshold.
- **¿Qué le paso al worker?** Task description destilada + context filtrado + output format + constraints + return reason. NUNCA la conversación cruda.
- **¿Voting es fair?** Solo si los agentes son genuinamente diversos. Si están entrenados parecido, voting es ECO, no robustez.
