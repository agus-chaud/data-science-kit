# Chuleta — Hito 2: RAG & MCP

> Referencia rápida. Para aprender de cero, andá a `milestones/02-rag-mcp.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es (1 línea) | El one-liner senior |
|---|---|---|
| `chunking-strategy` | Cómo partís el doc en chunks | "La pregunta NUNCA es 'qué chunk_size', es 'qué unidad semántica responde a las queries de mi usuario'." |
| `embeddings` | Vectores densos que capturan significado | "Es la fundación de TODO RAG. Cambiar de modelo post-deploy = re-indexar todo." |
| `vector-search` | Similitud (cosine/dot) sobre índices ANN | "ANN es APROXIMADO. FAISS es librería, no DB." |
| `hybrid-retrieval` | BM25 (sparse) + dense, fusionados con RRF | "Dense capta significado, sparse capta identidad exacta. Hybrid es DEFAULT post-2024, no nice-to-have." |
| `re-ranking` | Cross-encoder reordena el top-K | "Retrieval da recall, re-ranking da precisión. Bi-encoder para retrieval, cross-encoder para re-rank." |
| `mcp-protocol` | Estándar abierto LLM↔tools (JSON-RPC) | "Es el USB-C de los LLMs. No es function calling: es discovery + ejecución entre apps y servidores." |

## Tradeoff principal del hito — vector DBs

| DB | Conviene | Contra |
|---|---|---|
| FAISS | prototipo, embedded | no persiste, sin metadata filtering real |
| pgvector | stack Postgres existente, <10M | performance limitada >10M |
| Qdrant | producción seria, filtros payload | más infra |
| Weaviate | multi-modal, hybrid de fábrica | curva de aprendizaje |
| Pinecone | empresa con presupuesto | $$$, vendor lock-in |
| Milvus | escala billones | infra pesada |

## Top 3 anti-patterns (con el fix en 1 línea)

1. "RAG funciona" basado en 5 queries a mano → golden dataset de 100-300 queries reales + suite de eval (RAGAS) + regression antes de deploy.
2. Pure dense para queries con IDs/códigos ("DNI 30.000.000") → hybrid retrieval BM25+dense, default no-discusión.
3. Chunks gigantes de 4k tokens "para tener contexto" → chunks de 256-1024 según embedding; si querés más contexto, traés MÁS chunks, no más grandes.

## La pregunta de entrevista que más cae

**Q:** Tu RAG devuelve resultados malos. ¿Qué hacés?
**A (esqueleto):**
- Auditá 20 queries fallidas leyendo los chunks recuperados — no adivines.
- Si la info NO está en los chunks pero sí en el doc → chunking (quedó partida/sin contexto).
- Si está en los chunks pero rankeada abajo → retrieval/re-ranking.
- Si los chunks ni matchean → embedding model.
- Sin esa auditoría, todo fix es adivinanza.

## Decisión rápida (cheat)

- **¿Hybrid o solo dense?** ¿Hay keywords exactas / IDs / códigos / jerga en las queries? → hybrid. ¿Todo semántico/parafraseable? → dense alcanza (BM25 agrega ruido).
- **¿Re-ranker sí o no?** Casi siempre sí (top-50→re-rank top-20→top-5 al LLM). NO si el retrieval inicial ya tiene el gold en top-5, o si el re-ranker es de otro dominio.
- **¿pgvector o Qdrant?** <10M vectores + stack Postgres → pgvector (un servicio menos). >50M o filtros payload complejos → Qdrant.
- **¿chunk_size?** No hay default universal — se decide midiendo recall@k sobre golden dataset. BGE rinde con 256-512, text-embedding-3 con 512-1024.
- **¿MCP o tool use directo?** Tool one-off dentro de tu app → tool use directo. Tools reutilizables cross-cliente → MCP server.
