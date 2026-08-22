# huyen-ai-engineering

Skill de Claude Code que actúa como **knowledge base** del libro *AI Engineering: Building Applications with Foundation Models* de Chip Huyen (O'Reilly).

---

## Qué es esta skill

No es un mentor activo — es una base de referencia. Se activa cuando preguntás algo LLM-shaped (evaluación, RAG, prompt engineering, agentes, finetuning, dataset engineering, optimización de inferencia, arquitectura de aplicaciones) o cuando nombrás a Claude/Anthropic/otro proveedor de LLM y hace falta responder con precisión en vez de memoria. Busca el capítulo relevante y lo lee antes de responder.

Cubre **10 capítulos** — fundamentos de foundation models, metodología de evaluación, prompt engineering, RAG y agentes, finetuning, dataset engineering, optimización de inferencia, arquitectura y feedback de usuario.

---

## Para quién es

Para vos si estás construyendo o evaluando sistemas con LLMs y necesitás el framework correcto (no la intuición) para justificar una decisión: qué métrica de evaluación usar, cuándo finetunear vs. prompt engineering, cómo diseñar un dataset de eval, qué mover a caché vs. batch en inferencia.

---

## Cómo usarla

Se activa sola en contexto LLM-shaped. Uso explícito:

```
/huyen-ai-engineering                  → carga frameworks core
/huyen-ai-engineering RAG              → busca y carga el capítulo sobre RAG
/huyen-ai-engineering ch05             → carga el capítulo 5 puntual
```

Ejemplos de trigger real: *"qué métrica de evaluación uso para un sistema de RAG"*, *"cuándo me conviene finetunear en vez de prompt engineering"*, *"cómo diseño un dataset de eval sin data leakage"*.

Triggers y reglas de activación completas: **ver `SKILL.md`**.

---

## Cómo modificar / extender

| Querés cambiar... | Editá... |
|---|---|
| Reglas de activación o triggers | `SKILL.md` |
| Contenido de un capítulo puntual | `chapters/chXX-*.md` |
| Cheatsheet de referencia rápida | `cheatsheet.md` |
| Mapa de conceptos y relaciones | `concept-map.md` |
| Definiciones de términos | `glossary.md` |
| Patrones de arquitectura documentados | `patterns.md` |

---

## Instalación

```bash
git clone https://github.com/agus-chaud/data-science-kit.git
cp -r data-science-kit/skills/huyen-ai-engineering ~/.claude/skills/
```

Windows (PowerShell):
```powershell
Copy-Item -Recurse data-science-kit\skills\huyen-ai-engineering $env:USERPROFILE\.claude\skills\
```

Verificá preguntando "qué es RAG según Huyen" — si responde citando el libro, está instalada.

---

## Skills relacionadas

- [`senior-ai-engineer-mentor`](../senior-ai-engineer-mentor/README.md) — mentor activo que usa esta knowledge base (y la de Ahmad) como material de fondo, con pedagogía y tracking de mastery encima.
- [`ahmad-30-agents`](../ahmad-30-agents/README.md) — segunda knowledge base, foco en construir los 30 agentes nombrados del libro de Ahmad.

---

## Créditos

- **Material base**: "AI Engineering: Building Applications with Foundation Models" — Chip Huyen (O'Reilly).
- **Skill generada**: 2026-07-20.
