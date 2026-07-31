# Chuleta — Hito 3: dbt

> Referencia rápida. Para aprender de cero, andá a `milestones/03-dbt.md`. Esto es para repasar en 2 minutos.
> **Verificá la versión del usuario antes de afirmar comportamiento** — dbt cambió sintaxis entre versiones.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `dbt-project-structure` | staging → intermediate → marts | "Una responsabilidad por capa: eso es lo que mantiene el proyecto navegable a los tres años." |
| `dbt-ref-lineage` | `ref()`/`source()` construyen el DAG | "El grafo es el producto. Cada referencia hardcodeada es un nodo perdido." |
| `materializations` | Cómo el `SELECT` se vuelve objeto | "Es una decisión de costo, no de estilo." |
| `incremental-models` | Procesar solo lo nuevo | "Cuatro decisiones: clave, lookback, updates/deletes, reconciliación. Omitir una es un bug esperando." |
| `dbt-tests-contracts` | El contrato observable del modelo | "El test de grano vale más que diez `not_null`." |
| `dbt-jinja-macros` | Jinja compila antes de que exista el SQL | "Dos tiempos que no se mezclan. `target/compiled/` es tu debugger." |

## Tradeoff principal del hito — materializaciones

| Materialización | Costo de build | Costo de consulta | Cuándo |
|---|---|---|---|
| `view` | Nulo | Alto | Staging, poco consultado |
| `table` | Alto | Bajo | Marts chicos/medianos muy consultados |
| `incremental` | Bajo tras la primera | Bajo | Tablas grandes append-only |
| `ephemeral` | Nulo | Se hereda al padre | Paso intermedio sin consumo |
| Materialized view / dynamic table | Continuo (serverless) | Muy bajo | Frescura sin orquestar; ojo con el costo permanente |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Incremental que filtra contra `MAX()` del destino → lookback medido contra la latencia real de la fuente + test de reconciliación.
2. `dbt run` en producción en vez de `dbt build` → los datos rotos se propagan porque nada frena la cadena.
3. Referencias hardcodeadas → `ref()`/`source()` sin excepciones, con regla de CI que lo verifique.

## La pregunta de entrevista que más cae

**Q:** El proyecto dbt tarda 3 horas. ¿Qué hacés?
**A (esqueleto):**
- Sacar el tiempo por modelo (no adivinar cuál es el lento).
- Los top consumidores de tiempo: ver si son `table` que podrían ser incrementales, con el volumen de cambio real como criterio.
- Revisar si hay modelos que nadie consulta — se deprecian, no se optimizan.
- Revisar el paralelismo (`threads`) y si el warehouse está bien dimensionado para la carga.
- Recién al final, tuning de SQL. El orden importa: la ganancia grande casi nunca está ahí.

## Decisión rápida (cheat)

- **¿Vista, tabla o incremental?** Comparar costo de reconstruir contra frecuencia de consulta. Empezá simple; pasá a incremental cuando el número lo justifique.
- **¿Qué estrategia incremental?** ¿Hay clave única confiable? → `merge`. ¿El dato es naturalmente particionado por fecha? → reemplazo de partición. ¿Eventos inmutables? → `append`.
- **¿Un test solo?** Unicidad sobre la clave del grano. Es el que evita la duplicación de métricas, el error más caro.
- **¿Dónde va esta lógica?** ¿Renombrar/castear? → staging. ¿Reutilizable entre modelos? → intermediate. ¿Consume el negocio? → mart.
- **¿El SQL sale raro?** `dbt compile` y leé `target/compiled/`. El 90% de los bugs "raros" se resuelven ahí.
- **¿Extraigo un macro?** Al tercer uso, nunca antes, y con nombre de intención de negocio.
