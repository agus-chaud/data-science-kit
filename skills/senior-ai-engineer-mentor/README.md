# senior-ai-engineer-mentor

Skill de Claude Code que te convierte en **mentor activo de AI Engineering** con voz de Senior / Solutions Architect.

---

## Qué es esta skill

Una skill que se activa automáticamente cuando hacés preguntas pedagógicas sobre AI Engineering y responde como un Senior AI Engineer Gentleman: Rioplatense voseo, pasión genuina por enseñar, foco en CONCEPTOS antes que CÓDIGO. Trackea tu mastery por concepto en engram (4 niveles: `unknown` → `explored` → `practiced` → `mastered`, con repaso espaciado SM-2), te ahorra repetir lo que ya sabés, te empuja en lo que te falta. Cuatro modos especializados: `interview`, `review`, `project`, `explain`.

Cubre **32 conceptos en 6 hitos** (fundamentos, RAG/MCP, async/costos, frameworks, multi-agente, producción) — catálogo canónico en `concepts.md`.

---

## Para quién es

Para vos: **AI Engineer en formación que apunta a entornos corporativos competitivos** (Argentina, LATAM, mercado global remoto). No es para hobbyistas — asume que el objetivo es ser EMPLEABLE en posiciones donde te van a hacer preguntas de tradeoffs, anti-patterns, costos y modos de falla, no de "syntax de LangChain".

---

## Filosofía

- **CONCEPTS > CODE**: si no entendés POR QUÉ existe ReAct, no importa que sepas escribirlo.
- **Mentor activo, no pasivo**: te interrumpe cuando te ve flojo, te empuja cuando estás listo para el próximo nivel.
- **Libro como gimnasio**: el libro de Imran Ahmad ("30 Agents Every AI Engineer Must Build") tiene los ejercicios. La skill aporta el criterio senior que el libro no tiene.
- **AI es herramienta, vos dirigís**: la skill no te programa el agente — te enseña a diseñarlo.

---

## Cómo usarla

Se activa sola con preguntas pedagógicas ("explicame", "qué es", "preparame para entrevista"...). Triggers, comandos (`/ai-mentor status`, `/ai-mentor next`, `interview {concepto}`, etc.) y comportamiento completo: **ver `SKILL.md`** — es la única fuente de verdad operativa.

---

## Cómo modificar / extender

| Querés cambiar... | Editá... |
|---|---|
| Reglas de activación o comandos | `SKILL.md` |
| Agregar/quitar conceptos del catálogo | `concepts.md` |
| Cómo enseña un hito específico | `milestones/0X-*.md` |
| Comportamiento de un modo | `modes/{modo}.md` |
| Protocolo de onboarding | `prompts/onboarding-bootstrap.md` |
| Rúbricas para pasar a `mastered` | `prompts/feynman-checks.md` |
| Probes diagnósticos | `prompts/diagnostic-probes.md` |
| Bank de preguntas de entrevista | `playbooks/interview-questions-bank.md` |
| Anti-patterns referenciados | `playbooks/anti-patterns.md` |
| Tabla de tradeoffs | `playbooks/tradeoffs.md` |
| Links externos (MCP spec, OpenAI docs, etc.) | `playbooks/external-references.md` |

---

## Créditos

- **Material base**: "30 Agents Every AI Engineer Must Build" — Imran Ahmad (Packt, 2026). Notebooks en `C:/Users/Dell/Agus/Ai Agents Imran Ahmad/30-Agents-Every-AI-Engineer-Must-Build`.
- **Skill author**: Agustín, con voz Gentleman (Senior Architect, GDE & MVP).
- **Persona base**: definida en `~/.claude/CLAUDE.md` global.
