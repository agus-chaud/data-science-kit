# senior-ai-engineer-mentor

Skill de Claude Code que te convierte en **mentor activo de AI Engineering** con voz de Senior / Solutions Architect.

---

## Qué es esta skill

Una skill que se activa automáticamente cuando hacés preguntas pedagógicas sobre AI Engineering y responde como un Senior AI Engineer Gentleman: Rioplatense voseo, pasión genuina por enseñar, foco en CONCEPTOS antes que CÓDIGO. Trackea tu mastery por concepto en engram (4 niveles: `unknown` → `explored` → `practiced` → `mastered`, con repaso espaciado SM-2), te ahorra repetir lo que ya sabés, te empuja en lo que te falta. Cinco modos especializados: `onboarding` (el que te calibra y **te enseña a usar los otros 4**), `interview`, `review`, `project`, `explain`.

Cubre **36 conceptos en 7 hitos** (Hito 0 integración + fundamentos, RAG/MCP, async/costos, frameworks, multi-agente, producción) — catálogo canónico en `concepts.md`.

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

### El modo `onboarding` — arrancá por acá

Es el **5to modo**, y el más importante la primera vez: es el que te explica cómo usar los otros 4. Sin él, la skill funciona igual pero vas a "descubrir" comandos a los ponchazos.

- **Se activa solo**: la primera vez que le hacés una pregunta pedagógica y no hay mastery state guardado en engram, corre automático (4 minutos) antes de responderte.
- **Se puede invocar a mano**: `/ai-mentor onboarding` — corre el mismo protocolo bajo demanda, sin borrar tu progreso. Usalo si engram falló la primera vez, si querés repasar el mapa de comandos, o si le estás mostrando la skill a un compañero de equipo.
- **Qué hace**: greeting → te pregunta tu misión (para qué lo usás — ej. "necesito esto para el TP de WPP en IAAN") → calibra tu nivel con 1 pregunta + 3 probes conceptuales → te muestra una **tabla resumen de cómo invocar cada modo** → guarda todo en engram para no repreguntarte nunca más.
- **Fuente**: `prompts/onboarding-bootstrap.md`.

Si en algún momento querés repetirlo desde cero (perdiendo el progreso guardado), usá `/ai-mentor reset` en vez de `onboarding`.

---

## Requisito: engram + gentle-ai (para que se guarde tu progreso)

La skill trackea tu mastery llamando a herramientas `mem_*` (`mem_save`, `mem_search`, etc.) que vienen de **engram**, un servidor MCP de memoria persistente. Sin engram corriendo, la skill igual te explica conceptos, pero **no se acuerda de vos entre sesiones** — te repregunta todo cada vez.

- **`engram`**: memoria persistente para agentes de IA. Guarda tus decisiones, tu nivel por concepto y tu misión en una base local, y se la sirve a Claude Code como herramientas MCP (`mem_save`, `mem_search`, `mem_context`...). Es lo que le permite a esta skill decir "ya viste esto, vamos al siguiente" en vez de arrancar de cero cada charla.
- **`gentle-ai`**: el instalador/orquestador que provisiona todo el entorno de Claude Code en una máquina nueva — agentes soportados, persona, skills, reglas de `CLAUDE.md` y, entre sus componentes, **engram**. Es la forma recomendada de dejar engram (y el resto del setup) andando de una sola pasada, en vez de instalar cada pieza suelta a mano.

**Instalación** (en la máquina donde vas a correr Claude Code):

```bash
gentle-ai install
```

Esto instala el stack completo (incluye el componente `engram`). Si ya tenés el resto del entorno armado y solo te falta la memoria:

```bash
gentle-ai install --components engram
```

Verificá que quedó activo pidiéndole a Claude: *"guardá en tu memoria que estoy arrancando el TP de WhatsApp"* — si confirma que lo guardó (o corre `mem_save` sin error), engram está funcionando.

Si no tenés el binario `gentle-ai` en tu máquina todavía, consultá con la cátedra o con quien te compartió este repo por el instalador — no es un paquete público en npm/pip, es una herramienta interna del stack.

---

## Cómo usarla para el TP1 de WPP (cátedra IAAN)

El TP1 ("Sistema de atención por WhatsApp con IA para una empresa real") es exactamente el caso que cubre el **Hito 0** de esta skill: integración práctica de un agente sobre una plataforma externa (Kapso conecta WhatsApp Business), con persistencia en base de datos (Supabase) y MCPs + skills de Claude como capa de inteligencia — el mismo patrón que documenta `milestones/00-tp-integracion.md`.

Orden recomendado, mapeado a las 5 etapas de la consigna:

1. **Antes de arrancar el relevamiento** (semana 1-2): corré el `onboarding` (automático o `/ai-mentor onboarding`) y contale a la skill tu misión real — ej. *"TP1 de WhatsApp en IAAN, cliente es {pyme/rubro}, entrego el {fecha}"*. Esto hace que `/ai-mentor next` y `project` prioricen los conceptos que este TP realmente necesita (webhooks, tool-calling, memoria de agente, Supabase) en vez de un temario genérico.
2. **Para diseñar la solución técnica** (semana 3): usá `project: {tu caso — turnos / pedidos / triage / catálogo / cobranzas}`. El modo `project` reconoce la forma "webhook + agente + base de datos externa" y precarga el scope de Hito 0 en vez de arrancar de cero — te arma fases, stack opinado y riesgos, mapeado a lo que ya sabés.
3. **Mientras programás con Kapso + Supabase MCP** (semana 3-4): preguntá con `explicame`/`no entiendo` cada vez que uses un concepto nuevo del stack — memoria conversacional (ventana + extracción de estado, §3.6 de la consigna), tool-calling con efecto real vs. texto libre (§3.7), niveles de autonomía y validación en código antes de guardar un compromiso (§3.8). La skill te explica el CRITERIO antes de que copies el patrón sin entenderlo — que es justo lo que pide el criterio de evaluación "¿pueden explicar qué hizo el modelo y qué hicieron ustedes?".
4. **Antes de tocar código sensible** (guardar un turno, un pedido, un cobro): usá `review` + tu código de la herramienta/tool que hace ese guardado. Te lo revisa quirúrgicamente contra los criterios de Hito 0 (validación en código, no solo en el prompt).
5. **Antes de la presentación final** (semana 5): `interview {concepto}` sobre los temas que más pesan en el criterio "Uso de Claude/MCPs/Skills" — MCP, tool-calling, gestión de memoria — así llegás con las respuestas ya ensayadas si la cátedra pregunta por qué decidiste algo.

**No es para el relevamiento de negocio en sí** (entrevista al cliente, cuantificar el problema): para esa parte, la cátedra recomienda `/office-hours`. El modo `project` de esta skill te deriva ahí solo si detecta que tu idea todavía no tiene scope claro — así no se pisan los dos.

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

## Instalación

```bash
git clone https://github.com/agus-chaud/data-science-kit.git
cp -r data-science-kit/skills/senior-ai-engineer-mentor ~/.claude/skills/
```

Windows (PowerShell):
```powershell
Copy-Item -Recurse data-science-kit\skills\senior-ai-engineer-mentor $env:USERPROFILE\.claude\skills\
```

Verificá que quedó activa preguntando "explicame qué es RAG" — si responde en modo mentor, está instalada.

---

## Skills relacionadas

Esta skill es el **mentor** (voz, pedagogía, tracking de mastery). Las knowledge bases de los libros que usa como gimnasio viven como skills separadas en este mismo repo:

- [`ahmad-30-agents`](../ahmad-30-agents/README.md) — libro fuente del catálogo de conceptos y los 30 agentes de práctica.
- [`huyen-ai-engineering`](../huyen-ai-engineering/README.md) — segunda fuente, cubre evaluación, RAG, finetuning y producción con otro ángulo.

Usalas juntas: el mentor te interroga y trackea progreso, las knowledge bases responden preguntas puntuales de contenido cuando el mentor no está activado.

---

## Créditos

- **Material base**: "30 Agents Every AI Engineer Must Build" — Imran Ahmad (Packt, 2026). Notebooks en `C:/Users/Dell/Agus/Ai Agents Imran Ahmad/30-Agents-Every-AI-Engineer-Must-Build`.
- **Skill author**: Agustín, con voz Gentleman (Senior Architect, GDE & MVP).
- **Persona base**: definida en `~/.claude/CLAUDE.md` global.
