# Chuleta — Hito 1: Fundamentos

> Referencia rápida. Para aprender de cero, andá a `milestones/01-fundamentals.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `de-lifecycle` | Generación → ingesta → almacenamiento → transformación → serving | "Tu trabajo es gestionar los contratos entre etapas, no solo el código del medio." |
| `dimensional-modeling` | Hechos + dimensiones + grano declarado | "Si no podés escribir el grano en una oración, no entendiste el requerimiento." |
| `columnar-storage` | Datos guardados por columna | "Explica por qué `SELECT *` te sale caro y por qué los archivos chicos son veneno." |
| `batch-vs-streaming` | Frecuencia vs costo vs complejidad | "La pregunta nunca es batch o streaming: es cuánto cuesta que el dato tenga N minutos de atraso." |
| `idempotency-backfill` | Reejecutar seguro | "La fecha viene de afuera, nunca de adentro." |
| `table-formats` | Metadata que convierte archivos en tablas | "Sin table format tenés un directorio, no una tabla." |

## Tradeoff principal del hito — estrategia de escritura

| Estrategia | Idempotente | Cuándo |
|---|---|---|
| `INSERT` puro | ❌ | Nunca en producción |
| `MERGE` por clave | ✅ | Default con clave única real |
| Delete + insert de partición | ✅ | Datos particionados por fecha |
| `INSERT OVERWRITE` de partición | ✅ | Lake, partición como unidad natural |
| Append + dedupe en lectura | ✅ | Eventos inmutables de alto volumen |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Mart sin grano declarado → declarar el grano en una oración + test de unicidad sobre esa clave.
2. `CURRENT_DATE` adentro de la transformación → la ventana entra como parámetro del orquestador.
3. ELT que descarta el payload crudo → capa raw inmutable; el storage es lo más barato del stack.

## La pregunta de entrevista que más cae

**Q:** Hay que reprocesar 6 meses de un mart. ¿Cómo lo encarás?
**A (esqueleto):**
- Verificar que el pipeline sea idempotente — si no, primero se arregla eso.
- Verificar que el crudo de esos 6 meses todavía exista; si no, el backfill es imposible y ESA es la conversación.
- Reprocesar por particiones (mes o día), no todo junto, con límite de concurrencia.
- Validar conteos por chunk contra la fuente.
- Si el mart tiene consumidores activos, correr contra un esquema paralelo antes de tocar producción.

## Decisión rápida (cheat)

- **¿Star schema u OBT?** Más de un consumidor con preguntas distintas → star. Un solo consumidor → OBT es defendible.
- **¿SCD Type 1 o 2?** ¿Alguien puede nombrar la pregunta de negocio que necesita el histórico? Sí → Type 2. No → Type 1.
- **¿Batch o streaming?** ¿Hay una acción automática con ventana corta? Sí → evaluar streaming. No → batch, y punto.
- **¿Table format o tablas nativas?** ¿Varios motores o portabilidad real? Sí → Iceberg/Delta. No → nativas, menos operación.
- **¿Es idempotente?** Mirá dos cosas: de dónde sale la fecha, y cómo escribe. Si la fecha viene de adentro o escribe con `INSERT`, no lo es.
