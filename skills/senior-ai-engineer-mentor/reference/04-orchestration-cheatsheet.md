# Chuleta — Hito 4: Orquestación

> Referencia rápida. Para aprender de cero, andá a `milestones/04-orchestration.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `langchain-basics` | Composición de chains con LCEL (`prompt \| llm \| parser`) | "Junior elige por defecto, senior elige por contexto. 1-2 calls simples NO necesitan LangChain." |
| `langgraph-dags` | Agentes como grafos de estado explícitos | "Es una state machine explícita. El valor central es observabilidad y control." |
| `state-management` | Modelar/propagar el state entre nodos | "State es el CONTRATO entre nodos. State ≠ memory ≠ context window." |
| `checkpointing` | Persistir state por step | "Non-negotiable en producción. Habilita resume, HITL, time-travel debugging." |
| `llamaindex-vs-langchain` | Retrieval-first vs orquestación general | "No son rivales, se combinan: LlamaIndex como retriever DENTRO de un agente LangGraph." |

## Tradeoff principal del hito — qué framework usar

| Framework | Brilla en | Es overkill en |
|---|---|---|
| LangChain (LCEL) | composición de chains, multi-vendor swap, retrievers | 1-2 LLM calls simples |
| LangGraph | branching, HITL, state persistente | workflows lineales puros |
| LlamaIndex | RAG sofisticado, doc parsing, query engines | apps no-RAG-céntricas |
| Sin framework (raw SDK) | casos simples, control total, MVP | composición compleja repetida |
| CrewAI / AutoGen | prototipos multi-agent rápidos | producción seria |

## Top 3 anti-patterns (con el fix en 1 línea)

1. LangChain para llamar 1 vez a OpenAI → SDK directo; LangChain solo con composición real (≥3 calls con lógica).
2. State libre como dict sin schema → TypedDict/Pydantic con campos explícitos + reducers donde hay paralelismo.
3. Cero checkpointing, todo en memoria del proceso → PostgresSaver/RedisSaver desde día 1; thread_id por conversación.

## La pregunta de entrevista que más cae

**Q:** Workflow de 8 nodos secuenciales. ¿LangGraph o función Python con 8 awaits?
**A (esqueleto):**
- Si es PURAMENTE secuencial, sin branching/HITL/resume → función Python alcanza.
- LangGraph agrega valor con: branching condicional, parallel nodes con merge, HITL, checkpointing necesario, observabilidad en LangSmith.
- La trampa: usar LangGraph "porque queda bien en el CV". Es ceremonia si nada de eso aplica.

## Decisión rápida (cheat)

- **¿LangChain, LangGraph, LlamaIndex o nada?** Peso del problema en orquestación → LangChain/LangGraph. Peso en RAG (chunking complejo, query engines) → LlamaIndex. 50/50 → combinar. 1-2 calls → raw SDK.
- **¿Necesito reducer?** Si dos nodos paralelos escriben al mismo campo → sí (sin reducer = last-write-wins = perdés datos). Ej: `Annotated[list, add_messages]`.
- **¿Qué checkpointer?** Dev → MemorySaver. Single-machine → SqliteSaver. Producción multi-instance → PostgresSaver. Alta concurrencia + TTL → RedisSaver.
- **HITL en 5 pasos:** nodo draft → `interrupt_before=["execute"]` → checkpoint con thread_id → humano approve/reject → `update_state` + `invoke(None)` para resumir.
- **¿CrewAI/AutoGen?** Para producción seria, evitalos (falta madurez en observabilidad/error handling). LangGraph cubre lo mismo con más control.
