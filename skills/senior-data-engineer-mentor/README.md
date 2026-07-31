# senior-data-engineer-mentor

Skill de Claude Code que te convierte en **mentor activo de Data Engineering y APIs** con voz de Senior /
Solutions Architect.

---

## Qué es esta skill

Se activa automáticamente cuando hacés preguntas pedagógicas sobre data engineering y responde como un
Senior Data Engineer Gentleman: Rioplatense voseo, pasión genuina por enseñar, foco en CONCEPTOS antes que
CÓDIGO. Trackea tu mastery por concepto en engram (4 niveles: `unknown` → `explored` → `practiced` →
`mastered`, con repaso espaciado SM-2), te ahorra repetir lo que ya sabés y te empuja en lo que te falta.
Cuatro modos especializados: `interview`, `review`, `project`, `explain`.

Cubre **36 conceptos en 6 hitos** — catálogo canónico en `concepts.md`:

| Hito | Foco |
|---|---|
| 1 | Fundamentos: lifecycle, modelado dimensional, columnar, batch vs streaming, idempotencia, table formats |
| 2 | Snowflake: arquitectura, micro-particiones, warehouses, cachés, clustering, costo |
| 3 | dbt: estructura, ref/lineage, materializaciones, incrementales, tests/contracts, Jinja |
| 4 | Airflow: arquitectura, scheduling, TaskFlow, idempotencia, assets, escalado |
| 5 | APIs & MCP: diseño REST, OpenAPI, paginación, auth, confiabilidad, MCP |
| 6 | System Design & Delivery: split back/front, serving layer, Azure Pipelines, CI/CD de datos, IaC, gobierno |

---

## Para quién es

Para el que **ya usa estas herramientas todos los días y no las termina de entender**. No es un curso desde
cero: asume que escribís SQL, que corrés modelos y que mirás DAGs. Lo que aporta es el modelo mental de qué
pasa adentro — para que puedas PREDECIR el comportamiento de la herramienta, no solo invocarla.

Y asume un objetivo profesional: ser empleable en posiciones donde te van a preguntar por tradeoffs,
anti-patterns, costo y modos de falla, no por la sintaxis de un `MERGE`.

---

## Filosofía

- **CONCEPTS > CODE**: si no entendés por qué existe el pruning, no importa que sepas escribir el `WHERE`.
- **Abrir la caja negra**: cada explicación te tiene que dejar pudiendo predecir, no solo ejecutar.
- **El laburo es el gimnasio**: no hay ejercicios de juguete. Cada concepto tiene una tarea real de tu trabajo que sirve como evidencia para subir de nivel.
- **Todo tiene una factura**: en este dominio, una recomendación sin costo estimado está incompleta.
- **Mentor activo, no pasivo**: te interrumpe cuando te ve flojo, te empuja cuando estás listo.

---

## Cómo usarla

Se activa sola con preguntas pedagógicas ("explicame", "qué es", "por qué mi query es tan cara",
"preparame para entrevista"...). Triggers, comandos (`/de-mentor status`, `/de-mentor next`,
`interview {concepto}`, etc.) y comportamiento completo: **ver `SKILL.md`** — es la única fuente de verdad
operativa.

---

## Estado de las fuentes

Toda la base de conocimiento está construida sobre **documentación oficial verificada** (Snowflake, dbt,
Airflow, Microsoft Learn, Google API Design Guide, RFCs, specs de Parquet/Iceberg/OpenAPI/MCP). El catálogo
completo con qué leer de cada una está en `playbooks/external-references.md`.

**📕 Pendiente**: los tres libros canónicos del dominio (*Fundamentals of Data Engineering*, *DDIA*,
*The Data Warehouse Toolkit*) NO están cargados. Los conceptos que dependen de ellos están marcados
`📕 pendiente` en `concepts.md`, se enseñan con sustitutos públicos, y la skill tiene la obligación de
**declarar explícitamente** qué parte no está verificada. Si se aportan los PDF, se destilan parafraseados
y se saca la marca.

**Chequeo de versión**: dbt y Airflow cambian comportamiento entre versiones. La skill tiene la regla de
verificar la versión del usuario antes de afirmar comportamiento en los Hitos 3 y 4.

---

## Cómo modificar / extender

| Querés cambiar... | Editá... |
|---|---|
| Reglas de activación o comandos | `SKILL.md` |
| Agregar/quitar conceptos del catálogo | `concepts.md` (y el mapa de hitos de `SKILL.md`) |
| Cómo enseña un hito específico | `milestones/0X-*.md` |
| Comportamiento de un modo | `modes/{modo}.md` |
| Protocolo de onboarding | `prompts/onboarding-bootstrap.md` |
| Probes diagnósticos | `prompts/diagnostic-probes.md` |
| Rúbricas para pasar a `mastered` | `prompts/feynman-checks.md` |
| Banco de preguntas de entrevista | `playbooks/interview-questions-bank.md` |
| Catálogo de anti-patterns | `playbooks/anti-patterns.md` |
| Tablas de decisión de stack | `playbooks/tradeoffs.md` |
| Fuentes oficiales y comunidades | `playbooks/external-references.md` |
| Términos canónicos | `reference/GLOSSARY.md` |
| Chuletas de repaso | `reference/0X-*-cheatsheet.md` |

---

## Créditos

- **Estructura base**: `senior-ai-engineer-mentor`, misma arquitectura de mastery, modos y persistencia.
- **Fuentes**: documentación oficial de los vendors y specs públicas — ver `playbooks/external-references.md`.
- **Skill author**: Agustín, con voz Gentleman (Senior Architect, GDE & MVP).
- **Persona base**: definida en `~/.claude/CLAUDE.md` global.
