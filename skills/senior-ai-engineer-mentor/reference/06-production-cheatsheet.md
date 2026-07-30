# Chuleta — Hito 6: Producción & Governance

> Referencia rápida. Para aprender de cero, andá a `milestones/06-production.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `evals` | Golden dataset + métricas + regression CI | "Infra non-negotiable. Sin esto, mejorás a ciegas." |
| `observability` | Tracing de cada step + metrics + logs | "Distinta del backend tradicional: cada step tiene tokens, costo y contexto." |
| `safety-prompt-injection` | Defensas contra injection directa/indirecta | "El system prompt NO es boundary técnica. Defensa en capas o nada." |
| `compliance-argentina` | Ley 25.326 + Disp 2/2023 AAIP + Dto 836/2024 | "Aplica al RESPONSABLE, no al lugar de procesamiento. Que corra en US no te salva." |
| `compliance-global` | EU AI Act, GDPR, NIST, ISO 42001, sectoriales | "GDPR (datos) ≠ EU AI Act (riesgo del sistema). Podés cumplir uno y violar el otro." |
| `cost-attribution` | Costo LLM por tenant/feature/user | "Infra fundamental en B2B: sin esto, ni cobrás fair ni optimizás." |

## Tradeoff principal del hito — qué compliance según mercado

| Mercado | Marco aplicable | Esfuerzo |
|---|---|---|
| Solo AR | Ley 25.326 + Disp 2/2023 AAIP | bajo-medio |
| EU clientes | GDPR + EU AI Act (según risk class) | alto |
| US enterprise B2B | SOC2 Type II + ISO 42001 plus | alto |
| Healthcare US | HIPAA + BAA con providers | muy alto |
| Multi-jurisdicción | todos los que apliquen | muy alto, equipo dedicado |

## Top 3 anti-patterns (con el fix en 1 línea)

1. "Tenemos evals" = 10 queries a mano en notebook → golden dataset 100+, métricas múltiples, regression en CI, online evals con sampling humano.
2. "Defensa contra injection" = system prompt diciendo "no obedezcas al user" → defensa en capas (input filter + output validation + tool sandboxing + dual LLM + HITL + monitoring).
3. Costo unattributable ("nos cobraron 15k y no sabemos por qué") → metadata tagging desde día 1 + dashboard por dimensión + alertas.

## La pregunta de entrevista que más cae

**Q:** Tu sistema AI selecciona candidatos para empleos en EU. ¿Qué marcos aplican?
**A (esqueleto):**
- **EU AI Act:** empleo está en Annex III → HIGH-RISK. Risk management system, technical doc, human oversight obligatorio (no decisión 100% automatizada), conformity assessment, registro en EU database.
- **GDPR:** base legal, art 22 (decisiones automatizadas), DPIA obligatorio.
- Documentación: model card, evaluación de bias por categorías protegidas, audit log.
- Sin esto: multas brutales + prohibición de operar en EU.

## Decisión rápida (cheat)

- **¿Aplica Ley 25.326 si el LLM corre en US?** Sí. Aplica al responsable del tratamiento. Necesitás base legal + base de transferencia internacional (consent o cláusulas/DPA).
- **¿Anonimizar resuelve todo?** Solo si es REAL (no reversible, no re-identificable). Pseudonimización (reversible) sigue siendo dato personal según AAIP.
- **¿Langfuse o LangSmith?** OSS/self-host/multi-vendor → Langfuse. Stack LangChain + UI pulida y pagás → LangSmith.
- **¿Por qué el system prompt no defiende injection?** Porque el LLM no tiene boundary técnica entre system y user content: son tokens contiguos en el mismo contexto.
- **¿LLM-as-judge confiable?** Solo si: judge ≠ generador, validado contra labels humanos, criterios numéricos en el prompt. Mismo modelo = sesgo positivo brutal.
- **Multi-tenant cost:** per-tenant rate limits + budgets + queue prioritization + circuit breaker + multi-key isolation para tenants grandes.
