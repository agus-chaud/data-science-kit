# Chuleta — Hito 2: Snowflake

> Referencia rápida. Para aprender de cero, andá a `milestones/02-snowflake.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `snowflake-architecture` | Storage / compute / cloud services separados | "Preguntate siempre qué capa toca una operación: solo el compute se cobra por segundo." |
| `micro-partitions` | Bloques inmutables con metadata de rangos | "El pruning ES la optimización. Todo lo demás es secundario." |
| `virtual-warehouses` | Clusters de compute, cobrados por tiempo encendido | "Un warehouse es un grifo de plata que vos abrís y cerrás." |
| `snowflake-caching` | Result / metadata / local de warehouse | "El cache es medición contaminada: medí bytes, no segundos." |
| `clustering-pruning` | Co-localizar datos para mejorar el descarte | "Es la última palanca, no la primera." |
| `snowflake-cost` | Compute + storage + serverless | "La atribución es prerequisito de la optimización." |

## Tradeoff principal del hito — qué hacer según el síntoma

| Síntoma | Acción | Lo que NO hay que hacer |
|---|---|---|
| Query lenta con spilling | Scale up (warehouse mayor) | Agregar clusters |
| Queries en cola, cada una rápida | Scale out (multi-cluster) | Agrandar el warehouse |
| Costo alto sin uso | Bajar `AUTO_SUSPEND` | Achicar el warehouse |
| Escaneo del 100% de particiones | Arreglar el predicado | Poner clustering key |
| Un equipo afecta a otro | Separar warehouses | Subir el tamaño |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Warehouse con auto-suspend alto o apagado → suspender agresivo; es la fuga de plata más grande y la de menor riesgo.
2. Un solo warehouse para toda la empresa → uno por carga, para poder atribuir y dimensionar.
3. Clustering key como bala de plata → primero el predicado, después el orden de carga; clustering al final y con números.

## La pregunta de entrevista que más cae

**Q:** La factura de Snowflake se duplicó. ¿Qué hacés?
**A (esqueleto):**
- Comparar el consumo por warehouse mes contra mes: eso ubica QUÉ creció y de qué equipo es.
- Con el warehouse identificado, ir al historial de queries filtrado por ese warehouse y ordenar por tiempo total.
- Buscar tres causas típicas: un pipeline que empezó a correr más seguido, un modelo que pasó de incremental a full, o un warehouse que quedó encendido.
- Revisar en paralelo si alguien cambió tamaños o auto-suspend.
- Nunca empezar tuneando queries: primero ubicar el origen.

## Decisión rápida (cheat)

- **¿Scale up o scale out?** ¿La query individual es lenta? → up. ¿Las queries esperan en cola? → out.
- **¿Un warehouse grande sale más caro?** Solo si la carga NO escala. Se cobra por tiempo: el doble de recursos en la mitad del tiempo cuesta lo mismo.
- **¿Vale la pena la clustering key?** Solo si la tabla es grande, el patrón de filtro es estable, y el ahorro de compute supera el costo de reclustering. Medí las dos columnas.
- **¿Cómo mido si optimicé de verdad?** Bytes escaneados y particiones podadas. El cronómetro te miente por el result cache.
- **¿Cuánto Time Travel?** Por tabla, según valor de recuperación. Derivadas y reconstruibles: poco. Raw irrecuperable: más.
- **¿Por qué escanea todo?** Buscá una función o un cast encima de la columna filtrada. Es la causa número uno.
