# ahmad-30-agents

Skill de Claude Code que actúa como **knowledge base** del libro *30 Agents Every AI Engineer Must Build* de Imran Ahmad, PhD (Packt, 2026).

---

## Qué es esta skill

No es un mentor activo — es una base de referencia. Se activa cuando aplicás frameworks del libro (cognitive loop, paradigmas reactivo/deliberativo/híbrido, Agentic AI Progression Framework, PTCF prompting, pipelines RAG, orquestación de tools, multi-agente, protocolos MCP/A2A, agentes éticos y explicables, POMDP, TDG) o cuando construís cualquiera de los 30 agentes nombrados del libro (decisión autónoma, planificación, memoria, retrieval, healthcare, financiero, educación, visión-lenguaje, embodied, etc.).

Cubre **16 capítulos + epílogo** y es el catálogo canónico de conceptos que usa `senior-ai-engineer-mentor` como gimnasio de práctica.

---

## Para quién es

Para vos si estás construyendo un agente concreto y necesitás el patrón correcto del libro (no reinventarlo): qué arquitectura cognitiva usar, cómo estructurar el tool orchestration, cuándo pasar de single-agent a chain-of-agents, cómo diseñar el knowledge tracing de un agente educativo.

---

## Cómo usarla

Se activa sola en contexto de diseño/construcción de agentes. Uso explícito:

```
/ahmad-30-agents                       → carga frameworks core
/ahmad-30-agents RAG                   → busca y carga el capítulo relevante
/ahmad-30-agents tool orchestration    → busca por tema
/ahmad-30-agents ch06                  → carga el capítulo 6 puntual
```

Ejemplos de trigger real: *"armá el agente de Healthcare Intelligence del capítulo 13"*, *"qué paradigma de agente uso para un caso reactivo simple"*, *"cómo estructuro el chain-of-agents orchestrator"*.

Triggers y reglas de activación completas: **ver `SKILL.md`**.

---

## Cómo modificar / extender

| Querés cambiar... | Editá... |
|---|---|
| Reglas de activación o triggers | `SKILL.md` |
| Contenido de un capítulo puntual | `chapters/chXX-*.md` |
| Índice de los 30 agentes nombrados | `agents-index.md` |
| Cheatsheet de referencia rápida | `cheatsheet.md` |
| Mapa de conceptos y relaciones | `concept-map.md` |
| Definiciones de términos | `glossary.md` |
| Patrones de arquitectura documentados | `patterns.md` |

---

## Instalación

```bash
git clone https://github.com/agus-chaud/data-science-kit.git
cp -r data-science-kit/skills/ahmad-30-agents ~/.claude/skills/
```

Windows (PowerShell):
```powershell
Copy-Item -Recurse data-science-kit\skills\ahmad-30-agents $env:USERPROFILE\.claude\skills\
```

Verificá preguntando "qué agentes cubre el capítulo 13" — si responde citando el libro, está instalada.

---

## Skills relacionadas

- [`senior-ai-engineer-mentor`](../senior-ai-engineer-mentor/README.md) — mentor activo que usa este libro como catálogo canónico de conceptos (36 conceptos, 7 hitos) con pedagogía y tracking de mastery encima.
- [`huyen-ai-engineering`](../huyen-ai-engineering/README.md) — segunda knowledge base, complementa con evaluación, RAG y producción desde otro ángulo.

---

## Créditos

- **Material base**: "30 Agents Every AI Engineer Must Build" — Imran Ahmad, PhD (Packt, 2026).
- **Skill generada**: 2026-05-19.
