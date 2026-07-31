# Chuleta — Hito 4: Airflow

> Referencia rápida. Para aprender de cero, andá a `milestones/04-airflow.md`. Esto es para repasar en 2 minutos.
> **Verificá la versión antes de afirmar comportamiento** — Airflow 2 y 3 difieren en scheduling y en ejecución.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `airflow-architecture` | Scheduler / executor / workers / metadata DB / triggerer | "Dos tiempos: el archivo se PARSEA seguido, la tarea se EJECUTA una vez. Lo caro va adentro de la tarea." |
| `dag-scheduling` | Corridas dueñas de una ventana de datos | "Airflow no es un cron. Cada corrida es dueña de una ventana." |
| `operators-hooks-taskflow` | Plantillas de tarea, conexiones, TaskFlow, XCom | "Por XCom viajan referencias, no datos." |
| `dag-idempotency` | Reejecutar produce lo mismo | "Un reintento tiene que ser gratis. Si da miedo, la tarea está mal escrita." |
| `airflow-assets` | Scheduling cuando el dato está listo | "El reloj es una suposición; el asset es un hecho." |
| `airflow-scaling` | Pools, límites de concurrencia, deferrables | "Los slots son el recurso escaso. Si tu worker usa CPU, estás trabajando en el lugar equivocado." |

## Tradeoff principal del hito — cómo encadenar DAGs

| Patrón | Confiabilidad | Costo | Nota |
|---|---|---|---|
| Horario "con margen" | ❌ Baja | Nulo | Falla silenciosa cuando el upstream se atrasa |
| Sensor esperando al otro DAG | Media | Ocupa slot salvo deferrable | Acopla y puede quedar colgado |
| Trigger explícito del downstream | Alta | Bajo | Acopla productor a consumidor por nombre |
| Scheduling por asset | Alta | Bajo | Desacopla: el productor declara, el consumidor se suscribe |
| Un solo DAG | Alta | Bajo | Simple, pero mezcla ownership de equipos |

## Top 3 anti-patterns (con el fix en 1 línea)

1. DAGs encadenados por horario → dependencia por asset, trigger explícito, o unificar. Nunca por reloj.
2. `catchup=True` con `start_date` viejo → catchup desactivado por defecto y backfills explícitos y acotados.
3. Sensores no diferidos llenando la instancia → deferrable, o modo que libere el worker, más timeout siempre.

## La pregunta de entrevista que más cae

**Q:** "Airflow está lento" y agregar workers no mejoró nada. ¿Qué pasa?
**A (esqueleto):**
- Si sumar capacidad no cambia nada, el cuello no es de cómputo.
- Contar cuántas tareas ocupan slots sin trabajar: sensores esperando, tareas colgadas sin timeout.
- Revisar el scheduler: archivos de DAG pesados hacen lento el ciclo de parseo y frenan todo.
- Revisar la metadata DB: XComs gordos y retención infinita la hunden.
- Agregar workers a una instancia llena de tareas dormidas es pagar más asientos en una sala de espera.

## Decisión rápida (cheat)

- **¿Qué ventana procesa esta corrida?** La anterior. La corrida se dispara cuando el intervalo cierra.
- **¿Es idempotente?** Dos chequeos: ¿de dónde sale la fecha? ¿cómo escribe? Fecha del sistema o `INSERT` ciego → no lo es.
- **¿Esto va por XCom?** Solo si es un identificador, una ruta o una bandera. Nunca datos.
- **¿Dónde pongo este código?** Si tiene costo o efecto, adentro de la tarea. Nunca en el cuerpo del módulo.
- **¿Sensor o deferrable?** Toda espera de más de unos minutos, deferrable. Y timeout siempre.
- **¿Pool o más workers?** El pool protege sistemas de AFUERA. Más workers resuelve falta de capacidad real, que casi nunca es el problema.
