# Hito 2 — RAG + MCP

## Por qué importa (perspectiva corporativa)

Hermano, esto te lo digo en serio: el 90% de los proyectos LLM que pagan plata real en empresas son RAG. Punto. Mercado Libre con su buscador semántico de productos, Globant entregando RAG sobre documentación interna para clientes US, cualquier law-tech argentina haciendo búsqueda sobre fallos, fintech haciendo Q&A sobre regulaciones del BCRA — todo RAG. El que no sabe RAG en serio NO trabaja de AI Engineer en empresa que paga bien. Es así de fácil.

Y acá está el detalle que la mayoría no entiende: RAG NO es "pongo embeddings en una vector DB y listo". Eso es el tutorial de YouTube. RAG en producción es una **pipeline de 5-7 etapas** donde cada una tiene tradeoffs brutales: cómo chunkeás (¿semantic? ¿sentence-window? ¿hierarchical?), qué embedding model usás (¿OpenAI text-embedding-3-large carísimo o bge-large-v1.5 self-hosted?), retrieval híbrido o solo semántico, re-ranking sí o no, query expansion, citation tracking. Cada una de esas decisiones mueve calidad ±15% y costo ±5x.

MCP (Model Context Protocol) es el otro pilar de este hito y es 100% **el futuro inmediato**. Anthropic lo abrió end-2024, OpenAI lo adoptó en 2025, y para 2026 cualquier empresa seria con agentes lo está usando. Es el "USB-C" para conectar LLMs a herramientas y data sources. El AI Engineer que NO sabe MCP en 2026 está como un dev backend que en 2015 no sabía REST. Quedate sin trabajo, simple. Las oportunidades concretas: arquitecto de "RAG platform" interno en corporaciones (banca, salud, legal), AI Engineer en startups de búsqueda enterprise (Glean, Hebbia, Mendable), consultor de "tu RAG no funciona, vengo a arreglarlo" — esto último PAGA carísimo porque casi todos los RAG mal hechos están a producción rota.

## Conceptos de este hito

### chunking-strategy

**Qué es**: Cómo partís un documento largo en pedazos (chunks) que después se embedean e indexan. La decisión de tamaño, overlap, y boundaries (sentence, paragraph, semantic) determina qué tan bien recuperás contexto relevante.

**La trampa del junior**: `RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)` para todo. Bug feature ranking en clientes reales: chunks que cortan a la mitad de una tabla, dejan headers separados del contenido, parten una oración legal en dos. El RAG retrieva chunks "técnicamente similares" que NO tienen el contexto que el usuario necesita.

**Cómo lo piensa un senior**: Chunking se elige según **estructura del documento Y patrón de query**. Documento muy estructurado (legal, médico) → hierarchical chunking respetando secciones. Documento conversacional (transcripts, chats) → semantic chunking por cambio de tema. Documentos heterogéneos (PDFs con tablas, imágenes, texto) → **chunking multimodal por tipo** y indexar separado. La pregunta clave NUNCA es "qué chunk_size" — es "qué unidad semántica responde a las queries que mi usuario hace".

**Tradeoffs reales**:

| Strategy | Cuándo conviene | Costo dev | Pitfalls |
|---|---|---|---|
| Fixed-size + overlap | MVP, docs homogéneos | Mínimo | Corta mid-sentence, pierde estructura |
| Sentence-window | Q&A factual, docs prosa | Bajo | No respeta secciones |
| Recursive (LangChain default) | Genérico | Mínimo | "Default que nadie pensó" |
| Semantic chunking (embedding-based) | Docs con cambios de tema | Medio (compute upfront) | Frágil con docs cortos |
| Hierarchical (parent-child) | Docs muy estructurados, legal | Alto | Más índices, más complejidad |
| Late chunking (Jina) | Long context retrieval | Medio | Modelo embedding debe soportar |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué importa el chunk overlap?*
  A: Para que un concepto que cruza la frontera entre dos chunks pueda aparecer entero en al menos uno. Sin overlap, una oración relevante partida al medio puede no matchear ninguna query semántica. Típico 10-20% del chunk_size.
- Q (senior): *Tu RAG sobre PDFs legales devuelve chunks irrelevantes pero "similares". ¿Cómo diagnosticás chunking vs embedding vs retrieval?*
  A: Primero auditás 20 queries fallidas: leo los chunks retrieved. Si los chunks NO contienen la info pero el doc original sí, es chunking (la info quedó partida o sin contexto). Si los chunks tienen la info pero el ranking la pone abajo, es retrieval/re-ranking. Si los chunks ni siquiera matchean semánticamente, es embedding model. Sin esa auditoría, todos los fix son adivinanza.
- Q (trampa): *¿Conviene chunks de 256 tokens o 1024?*
  A: Trampa: no hay respuesta universal. Depende del modelo embedding (BGE rinde mejor con 256-512, text-embedding-3 con 512-1024), del tipo de query (factual corto vs analítico largo), y del LLM downstream (más contexto = más caro pero mejor síntesis). Un senior responde "lo decido midiendo recall@k sobre golden dataset, no por defaults".

### embeddings

**Qué es**: Modelos que convierten texto en vectores densos (768-3072 dimensiones típicamente) que capturan significado: textos similares en significado → vectores cercanos en espacio.

**La trampa del junior**: Usar `text-embedding-ada-002` (deprecated en 2024) porque "es el de OpenAI". O al revés: bajar el primer modelo de HuggingFace sin chequear si está fine-tuneado para tu dominio/idioma. Ignorar que español/portugués/legal/médico requieren modelos específicos.

**Cómo lo piensa un senior**: Embedding model es **la fundación de TODO RAG**. Decisión de elección incluye: (1) dimensionalidad (más dim = más calidad pero más storage y compute), (2) idioma/dominio (multilingual vs especializado), (3) latencia y throughput (API vs self-hosted), (4) costo (API por token vs GPU mensual), (5) MTEB scores en el dominio relevante. Cambiar de embedding model post-deploy es **re-indexar todo** — decisión cara de corregir.

**Tradeoffs reales**:

| Modelo | Dims | Costo | Cuándo |
|---|---|---|---|
| OpenAI text-embedding-3-small | 1536 (truncable) | $0.02/1M | Default razonable, multi-idioma decente |
| OpenAI text-embedding-3-large | 3072 (truncable) | $0.13/1M | Calidad alta, presupuesto OK |
| Cohere embed-multilingual-v3 | 1024 | $0.10/1M | 100+ idiomas serios |
| bge-large-en-v1.5 (BAAI) | 1024 | Self-hosted | Inglés, top MTEB, costo fijo |
| bge-m3 (BAAI) | 1024 | Self-hosted | Multi-idioma + multi-functional (dense+sparse+colbert) |
| jina-embeddings-v3 | 1024 | API/self-hosted | Multilingüe + tareas especializadas |
| voyage-3-large | 1024 | $0.18/1M | Top MTEB 2025, RAG-optimized |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué un vector de 1536 dimensiones representa significado?*
  A: Porque el modelo se entrenó (contrastive learning típicamente) para que pares de textos semánticamente cercanos terminen con vectores con alta cosine similarity, y pares lejanos con baja. Cada dimensión no es interpretable individualmente — es la geometría conjunta del espacio la que codifica significado.
- Q (senior): *Tenés un RAG en español sobre contratos. ¿Qué embedding model elegís y por qué?*
  A: Primer pass: voyage-3 o cohere-multilingual-v3 por calidad en español. Segundo pass: si presupuesto justo y volumen alto, bge-m3 self-hosted en una GPU L4 (sale ~300 USD/mes para throughput decente, vs ~2000 USD/mes en API a volumen alto). Tercer pass: si tengo recursos para fine-tune, mistral-embed o bge fine-tuneado en corpus legal AR sube recall significativo.
- Q (trampa): *¿Conviene siempre usar el embedding con más dimensiones?*
  A: NO. Más dim = más storage (3072 dim × 4 bytes × 10M chunks = 120GB solo de vectores), más compute en cada query, y rendimientos decrecientes después de cierto punto. Matryoshka embeddings (text-embedding-3) permiten truncar a 512 dim manteniendo 90% calidad — pattern moderno. La pregunta correcta es "qué dim minimiza costo manteniendo recall@k > X".

### vector-search

**Qué es**: Búsqueda por similitud (cosine, dot product) sobre índices ANN (Approximate Nearest Neighbor) como HNSW, IVF, ScaNN. Trade exactitud por velocidad para escalar a millones/billones de vectores.

**La trampa del junior**: Usar FAISS in-memory en notebook, mostrar el demo, deployar a producción con 10M docs y descubrir que (a) no persiste sin código extra, (b) no soporta filtros metadata bien, (c) re-indexar es offline. Crisis a los 3 meses.

**Cómo lo piensa un senior**: Vector DB choice se decide por **4 ejes**: persistencia, filtros metadata (pre vs post filtering), escalabilidad (single-node vs distributed), y operatoria (managed vs self-hosted). FAISS es **librería**, no DB — sirve para prototipo o caso embebido. Para producción serio: pgvector (si ya tenés Postgres), Qdrant (rust, performance + filtros), Weaviate (multi-modal, OSS), Pinecone (managed, caro), Milvus (escala billones).

**Tradeoffs reales**:

| DB | Pro | Contra | Cuándo |
|---|---|---|---|
| FAISS | Rápido, Free | No persiste, sin metadata filtering real | Prototipo, embedded |
| pgvector | Misma DB que app, transacciones, filtros SQL | Performance limitada >10M vectors | Stack Postgres existente, <10M |
| Qdrant | Performance + filtros + payload | Más infra | Producción seria, on-prem ok |
| Weaviate | Multi-modal nativo, OSS | Curva aprendizaje | Multi-modal, hybrid de fábrica |
| Pinecone | Managed, sin ops | $$$, vendor lock-in | Empresa con presupuesto |
| Milvus | Escala billones, distributed | Infra pesada | Big data scale |

**En entrevista te van a preguntar**:
- Q (mid): *Diferencia entre cosine similarity y dot product.*
  A: Cosine normaliza por magnitud — solo mide ángulo. Dot product es magnitud × ángulo. Si tus embeddings ya están L2-normalizados (la mayoría sí), son equivalentes y dot product es más rápido. Si NO están normalizados, cosine es lo correcto para "similitud semántica" pura.
- Q (senior): *Tu vector DB devuelve resultados rápidos pero a veces "fallan" — el doc que claramente matchea no está en top-10. ¿Qué pasa?*
  A: ANN es APROXIMADO. HNSW e IVF sacrifican exactitud por velocidad. Tres causas típicas: (1) parámetros del índice mal seteados (ef_search bajo en HNSW, nprobe bajo en IVF), (2) índice no rebalanceado tras muchos inserts, (3) curse of dimensionality si dim muy alto. Soluciones: subir ef_search (más latencia, más recall), o pre-filtering por metadata para reducir el espacio candidato.
- Q (trampa, system design): *¿pgvector o Qdrant?*
  A: Depende. Pregunta correcta: ¿cuántos vectores total tenés y vas a tener en 1-2 años? ¿Querés filtros relacionales complejos (joins) sobre metadata? ¿Tu equipo opera Postgres ya? Si <10M vectores y stack Postgres → pgvector gana (un servicio menos). Si >50M o filtros payload complejos → Qdrant. La trampa: elegir por "lo más cool" en vez de por restricciones operativas.

### hybrid-retrieval

**Qué es**: Combinar **keyword search** (BM25, sparse) con **semantic search** (dense vectors) y mergear con un algoritmo de fusión (Reciprocal Rank Fusion, weighted sum). Cubre queries que pura semántica no resuelve (códigos exactos, nombres propios, jerga técnica).

**La trampa del junior**: Usar solo dense embeddings porque "lo viejo (BM25) está obsoleto con LLMs". Después el usuario busca "DNI 27.000.000" o "ERR_CONNECTION_REFUSED" y el RAG NO encuentra nada — los embeddings no diferencian bien strings exactos raros.

**Cómo lo piensa un senior**: Dense y sparse capturan **señales distintas**. Dense capta significado parafraseable; sparse capta identidad exacta. Queries reales mezclan ambas (`"problemas con el error ERR_CONN en mi router"` mezcla concepto + token exacto). Hybrid retrieval no es "nice-to-have" — es **default** en RAG serio post-2024. El ratio dense/sparse típico es 0.6/0.4 pero se tunea por dominio.

**Tradeoffs reales**:

| Approach | Pro | Contra |
|---|---|---|
| Solo dense (vector) | Captura paráfrasis, multi-idioma | Falla en exact-match, jerga, IDs |
| Solo sparse (BM25) | Exact match, rápido, interpretable | No entiende sinónimos |
| Hybrid (RRF) | Mejor recall típicamente +15-25% | Más infra (2 indexes), tuning de pesos |
| Hybrid (weighted sum) | Control fino | Pesos dependen de scores normalizados — tricky |
| ColBERT / multi-vector | Late interaction, calidad alta | Storage 10-100x, latencia mayor |
| BGE-M3 (dense+sparse+ColBERT) | Todo en un modelo | Más compute por query |

**En entrevista te van a preguntar**:
- Q (mid): *¿Por qué BM25 sigue siendo útil con LLMs?*
  A: Porque maneja exact match y términos raros mejor que embeddings. Si el usuario busca un código de error específico, un número de factura, o jerga muy técnica, los embeddings tienden a "suavizar" eso y devolver resultados temáticamente similares pero no específicos. BM25 te lo da exacto.
- Q (senior): *¿Cómo combinás scores de dense y BM25 si están en escalas distintas?*
  A: Dos approaches estándar: (1) **Reciprocal Rank Fusion (RRF)**: ignorás scores, usás solo rankings — `score = Σ 1/(k + rank_i)`. Es robusto y no necesita normalizar nada. (2) **Weighted sum normalizado**: normalizás cada score a [0,1] con min-max o softmax y promediás con pesos α/(1-α). RRF es más simple y generalmente competitivo. Weighted sum es más tunable pero más frágil.
- Q (trampa): *Tu RAG con hybrid pesa 60% dense, 40% BM25. Performance baja vs solo dense. ¿Por qué?*
  A: Probablemente tu corpus es prosa pura (artículos, documentación conversacional) sin jerga ni exact-match queries. BM25 está agregando ruido. Solución: medí por TIPO de query — si las queries son 95% parafrasables, baja peso BM25 a 0.1 o 0. Hybrid no es siempre mejor — depende del dominio.

### re-ranking

**Qué es**: Segunda pasada sobre el top-K del retrieval inicial usando un modelo más caro (cross-encoder) que mira (query, chunk) juntos y reordena. Mejora precisión a costo de latencia.

**La trampa del junior**: Pedirle al LLM ("oye GPT-4, reordená estos 10 chunks por relevancia a la query"). Funciona pero cuesta 100x más que un re-ranker dedicado, es más lento, y no es reproducible.

**Cómo lo piensa un senior**: Retrieval da recall (cubre el espacio), re-ranking da precisión (ordena bien). Cross-encoders como bge-reranker-v2-m3 o cohere-rerank-3 procesan (query, doc) JUNTOS — captan interacción semántica fina que los bi-encoders (embeddings) no pueden. Costo: latencia ~50-150ms para top-20 reordenado. Vale la pena casi siempre en RAG production-grade.

**Tradeoffs reales**:

| Re-ranker | Latencia (top-20) | Costo | Calidad |
|---|---|---|---|
| Cohere Rerank 3 | ~80ms (API) | $1/1k searches | Top calidad multilingüe |
| Jina Reranker v2 | ~60ms | API/self-host | Buena y barata |
| bge-reranker-v2-m3 | ~100ms (GPU) | Self-host | Top OSS, multilingüe |
| MS MARCO MiniLM | ~30ms (CPU OK) | Self-host | Calidad media, rápido |
| LLM-as-reranker (GPT-4) | ~2s | $$$ | Calidad variable, no recomendado |
| Sin re-ranker | 0 | 0 | Baseline |

**En entrevista te van a preguntar**:
- Q (mid): *Diferencia entre bi-encoder y cross-encoder.*
  A: Bi-encoder embede query y doc por separado, después compara con cosine. Permite indexar docs offline. Cross-encoder los procesa juntos en un solo forward pass, no permite indexación, pero capta interacción semántica más rica. Por eso bi-encoder para retrieval (rápido sobre millones), cross-encoder para re-rank (preciso sobre top-K).
- Q (senior): *Tu retrieval devuelve top-100. ¿Re-rankeás los 100 o un subset?*
  A: Trade-off latencia vs cobertura. Típico: retrieval top-50 a 100, re-rank top-20 a 30, mandar top-5 a 10 al LLM. Re-rankear 100 con cross-encoder cuesta ~500ms — inaceptable para chat real-time. Si la query es batch/offline, podés ir más alto. La elección final viene de medir nDCG@10 vs latencia.
- Q (trampa): *¿Re-ranker siempre mejora?*
  A: Casi siempre, pero hay casos donde NO: (1) si tu retrieval inicial ya es muy bueno (top-5 ya tiene gold), el re-ranker solo agrega latencia sin ganancia; (2) si el re-ranker está entrenado en dominio muy distinto al tuyo (legal con re-ranker general), puede REORDENAR PEOR. Validá con golden dataset antes de meterlo en pipeline.

### mcp-protocol

**Qué es**: **Model Context Protocol** (Anthropic, open-source, late-2024). Estándar para conectar LLMs a herramientas, data sources y prompts via servidores MCP. JSON-RPC 2.0 sobre stdio/SSE. Resuelve la fragmentación de "cada LLM tiene su propio function calling".

**La trampa del junior**: Confundir MCP con function calling. NO ES LO MISMO. Function calling es **wire format** entre LLM y app (un mensaje "quiero ejecutar X"). MCP es **protocolo de descubrimiento + ejecución** entre apps y SERVIDORES de tools. MCP usa function calling por debajo, pero agrega: discovery dinámico, prompts compartidos, resources (data), capability negotiation.

**Cómo lo piensa un senior**: MCP es el **"USB-C de los LLMs"** — y lo digo en serio, no es marketing. Antes de MCP: cada integración tool↔LLM era custom (escribís tool para OpenAI, la re-escribís para Claude, otra para Gemini). Con MCP: escribís un MCP server UNA VEZ, cualquier cliente compatible (Claude Desktop, Cursor, Continue, custom apps) lo usa. El AI Engineer que NO sabe MCP en 2026 está como un dev backend en 2015 que no sabía REST.

**Tradeoffs reales**:

| Approach | Cuándo |
|---|---|
| Tool use directo (Anthropic/OpenAI) | Tool custom one-off dentro de tu app |
| MCP server local (stdio) | Tools que corren en la máquina del usuario (file system, git local) |
| MCP server remoto (SSE/HTTP) | Tools centralizados, multi-tenant, shared infra |
| LangChain Tools | Stack ya en LangChain, no necesitás portabilidad cross-cliente |
| Custom RPC | Casos muy específicos, control total — pero perdés ecosistema |

**En entrevista te van a preguntar**:
- Q (mid): *¿Qué es MCP y qué problema resuelve?*
  A: Protocolo abierto que estandariza cómo un cliente LLM se conecta a servidores que exponen tools, prompts y resources. Resuelve la fragmentación: antes cada integración era custom per-vendor, ahora un servidor MCP funciona con cualquier cliente compatible (Claude Desktop, Cursor, etc). Usa JSON-RPC 2.0.
- Q (senior): *Diferencia entre tools, prompts y resources en MCP.*
  A: **Tools** = funciones ejecutables (side effects, return value). **Resources** = data leíble identificada por URI (`file://...`, `db://...`), el cliente decide cuándo leerla. **Prompts** = templates parametrizables que el server expone al usuario. La separación importa: tools son acción, resources son contexto, prompts son UX guidance. Mezclarlos es smell de mal diseño.
- Q (trampa, security): *Un MCP server local tiene acceso al filesystem. ¿Cuál es el modelo de threat?*
  A: El MCP server corre con los privilegios del proceso que lo lanzó. Si el LLM puede invocar `tool: file_read(path)` con `path = "/etc/shadow"`, el server lo ejecuta. Defensas: (1) server debe implementar **allowlist** de paths, no confiar en args, (2) cliente debe pedir consent del usuario antes de exponer un server con permisos amplios (Claude Desktop hace esto), (3) principle of least privilege en el proceso del server, (4) audit log de toda invocación. El protocolo NO te da seguridad — vos implementás las policies.

## Lo que el libro hace bien acá

- **chapter06** — `Information Retrieval & Knowledge Agents` — implementa RAG con FAISS, chunking básico, y un agente Document Intelligence con OCR. Bueno para ver la pipeline RAG end-to-end sin frameworks. El notebook de Scientific Research muestra clustering de embeddings — útil para entender el espacio vectorial.
- **chapter02** — `The Agent Engineer's Toolkit` — nombra y compara vector DBs (FAISS, Pinecone, Weaviate, Qdrant). No profundiza pero da el lay-of-the-land.
- **chapter14** — `Financial & Legal Domain Agents` — RAG aplicado a contratos legales. Ojo: el chunking es básico, no llega a hierarchical. Tomalo como ejemplo de aplicación, no como referencia técnica de chunking avanzado.

## Lo que el libro NO tiene (gaps a saber)

- **MCP (Model Context Protocol)**: el libro lo menciona como `MCPRegistry` en chapter01 pero NO entra al protocolo real.
  - Recurso: https://modelcontextprotocol.io/specification + https://modelcontextprotocol.io/quickstart/server
  - Qué entender: estructura cliente/servidor, JSON-RPC 2.0 over stdio/SSE, distinción tools/resources/prompts, capability negotiation, security model. Implementá un MCP server tuyo siguiendo el quickstart — es la mejor forma de internalizar.

- **Hybrid retrieval (BM25 + semantic)**: el libro hace pure-dense. Gap importante.
  - Recurso: https://python.langchain.com/docs/integrations/retrievers/ensemble (LangChain EnsembleRetriever) + paper "Reciprocal Rank Fusion" (Cormack et al., 2009).
  - Qué entender: cuándo dense gana, cuándo sparse gana, fusion algorithms (RRF vs weighted sum), evaluación con golden dataset por tipo de query.

- **Re-ranking con cross-encoders**: ausente del libro.
  - Recurso: https://docs.cohere.com/docs/rerank-overview + https://huggingface.co/BAAI/bge-reranker-v2-m3
  - Qué entender: bi-encoder vs cross-encoder, por qué cross-encoder no se puede pre-indexar, cómo elegir top-K para re-rank vs top-K para LLM, métricas (nDCG@k, MRR).

- **Advanced chunking (semantic, hierarchical, late chunking)**: el libro hace fixed/recursive.
  - Recurso: https://docs.llamaindex.ai/en/stable/module_guides/loading/node_parsers/modules/ (LlamaIndex node parsers) + https://jina.ai/news/late-chunking-in-long-context-embedding-models/
  - Qué entender: cuándo cada strategy gana, costo computacional upfront, mantenibilidad cuando los docs cambian.

- **Evaluación de RAG (RAGAS, golden datasets)**: el libro no cubre cómo MEDIR un RAG.
  - Recurso: https://docs.ragas.io/ + Anthropic RAG eval cookbook
  - Qué entender: faithfulness, answer relevancy, context precision/recall, cómo construir un golden dataset de 100-300 ejemplos, regression suite por release.

## Ejercicios para subir de nivel

### Para subir a `practiced`

- `chunking-strategy`: corré `chapter06/notebook.ipynb`. Modificá el chunking de fixed-size a sentence-window (usá LlamaIndex `SentenceWindowNodeParser`). Compará retrieval sobre 5 queries de prueba. Pegame los chunks devueltos en ambos casos.
- `embeddings`: en `chapter06`, reemplazá el embedding por uno multilingual (bge-m3 o cohere-multilingual). Compará recall sobre 5 queries en español.
- `vector-search`: en `chapter06`, migrá de FAISS a Qdrant local (Docker). Mostrame el código de la migración y latencia comparada.
- `hybrid-retrieval`: NO hay notebook. Implementá un retriever que combina FAISS dense + rank_bm25 sparse con RRF. Pegame el código (~50 líneas) y un ejemplo de query donde hybrid gana a solo-dense.
- `re-ranking`: NO hay notebook. Agregá `bge-reranker-v2-m3` (HuggingFace) o cohere-rerank al top-10 de retrieval del chapter06. Mostrame el delta de ranking sobre 3 queries.
- `mcp-protocol`: NO hay notebook. Implementá un MCP server mínimo siguiendo el quickstart oficial (Python o TypeScript). Que exponga UNA tool (ej: `get_current_time(timezone)`). Conectalo a Claude Desktop. Pegame el código + screenshot funcionando.

### Para subir a `mastered`

- `chunking-strategy`: en un proyecto propio, diseñá la estrategia de chunking para un corpus real (no del libro). Justificá la elección midiendo recall@5 sobre 30 queries de prueba contra al menos otra strategy. Feynman check: explicáme en 3 oraciones por qué chunk_size depende del modelo embedding.
- `embeddings`: tomá un proyecto real. Calculá el costo total de embeddear el corpus + queries esperadas/año con 3 modelos distintos (OpenAI, Cohere, self-hosted). Defendé la elección con número.
- `vector-search`: deployá un vector DB en producción (managed o self-hosted) con persistencia, backup, monitoring. Documentá el setup. Defendé por qué elegiste ese stack.
- `hybrid-retrieval`: corré evaluación con RAGAS o golden dataset comparando dense vs hybrid en tu proyecto. Decidí pesos finales con datos.
- `re-ranking`: integrá re-ranker en pipeline real con SLA de latencia <500ms total. Reportá nDCG@10 antes vs después. Explicáme por qué cross-encoder no sirve para retrieval inicial.
- `mcp-protocol`: implementá un MCP server real que un equipo usaría (ej: server que expone tu DB de tickets, o tu confluence). Documentá el threat model y las defensas implementadas. Feynman check: explicale a un dev backend qué es MCP usando una analogía no-LLM.

## Anti-patterns que vas a ver en clientes reales

1. **"RAG funciona", basado en 5 queries hechas a mano**
   - Cómo se hace: dev arma RAG, prueba 5 queries que se le ocurren, "funciona", a producción.
   - Por qué se hace: no hay cultura de eval en LLM products todavía.
   - Costo real: 30-50% de queries reales fallan, usuarios dejan de usar, equipo "fixea" prompts en círculos sin data.
   - Cómo lo arregla un senior: golden dataset de 100-300 queries reales con respuestas esperadas, suite de eval (RAGAS o custom), regression check antes de cada deploy.

2. **Pure dense retrieval para queries con códigos/IDs**
   - Cómo se hace: "usamos OpenAI embeddings para todo".
   - Por qué se hace: "embeddings son lo nuevo, BM25 es lo viejo".
   - Costo real: usuarios buscando "DNI 30.000.000" o "código de error E_PAYMENT_DECLINED_017" no encuentran nada. Soporte se llena de tickets.
   - Cómo lo arregla un senior: hybrid retrieval con BM25 sparse + dense fusion. Default no-discusión en RAG corporativo.

3. **Re-indexar manualmente cuando cambian docs**
   - Cómo se hace: `python rebuild_index.py` cada vez que alguien edita un PDF.
   - Por qué se hace: scripts de pet project sin pipeline.
   - Costo real: índice queda stale, usuarios reciben info vieja, nadie sabe cuándo fue el último refresh.
   - Cómo lo arregla un senior: pipeline event-driven (webhook on doc change → enqueue → embed → upsert) o batch nocturno con monitoring. Track `last_indexed_at` por doc.

4. **MCP servers exponiendo todo sin allowlist**
   - Cómo se hace: MCP server con `tool: shell_exec(cmd)` o `tool: file_read(any_path)` sin restricciones.
   - Por qué se hace: "lo pruebo local, total".
   - Costo real: prompt injection del usuario / del documento RAG ejecuta comandos arbitrarios. Caso real esperable en 2026.
   - Cómo lo arregla un senior: allowlist de paths/comandos, validación de args, consent explícito del usuario, audit log.

5. **Chunks gigantes (4k tokens) para "tener contexto"**
   - Cómo se hace: `chunk_size=4000` porque "el LLM tiene contexto largo".
   - Por qué se hace: confunden chunking (retrieval) con context window (LLM input).
   - Costo real: cada chunk capta muchos temas → embeddings ruidosos → retrieval menos preciso → respuestas peores aunque el LLM tenga "mucho contexto".
   - Cómo lo arregla un senior: chunks de 256-1024 tokens según modelo embedding. Si necesitás más contexto, traés MÁS CHUNKS al LLM (top-10 vs top-3), no chunks más grandes.

## Checkpoint

Cuando podés contestar SÍ a estas preguntas, este hito está dominado:

- [ ] ¿Podés diseñar una pipeline RAG end-to-end para un dominio nuevo justificando cada decisión (chunker, embedding, vector DB, re-ranker) con tradeoffs?
- [ ] ¿Podés explicar por qué hybrid retrieval mejora sobre solo-dense, y cuándo NO mejora?
- [ ] ¿Sabés cuándo conviene re-ranker y cuándo es overhead inútil, con criterio cuantitativo?
- [ ] ¿Podés implementar un MCP server mínimo + conectarlo a un cliente + explicar el modelo de threat?
- [ ] ¿Podés defender tu elección de chunk_size con números (recall@k sobre golden dataset) en vez de "el default de LangChain"?
- [ ] En entrevista senior, ¿podés contestar "tu RAG devuelve resultados malos, ¿qué hacés?" con un protocolo de diagnóstico ordenado en vez de tirar fixes random?
