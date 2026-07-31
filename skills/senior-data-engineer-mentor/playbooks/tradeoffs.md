# Tradeoffs — tablas de decisión

**Cuándo se carga**: desde `modes/project.md` al proponer stack, o cuando el usuario pregunta "¿qué conviene
para X?".

**Cómo se usa**: cada tabla es una decisión con sus opciones. El mentor NO devuelve la tabla entera — elige
y **opina**, usando la tabla como respaldo. *"Depende"* sin recomendación no vale.

**Regla transversal de este dominio**: cada elección tiene una **factura mensual**. Si no podés estimar el
costo operativo de una opción, esa es una pregunta pendiente, no un detalle.

---

## Decisión: dónde transformar

| Opción | Conviene cuando | Contra | Costo |
|---|---|---|---|
| SQL en el warehouse (dbt) | Default. Datos ya en el warehouse, equipo con SQL | No sirve para lógica que SQL no expresa | Créditos de compute |
| Python en un contenedor | Lógica compleja, librerías específicas, APIs | Otro artefacto que construir y desplegar | Compute del runtime |
| Python en el worker del orquestador | ❌ Casi nunca | Convierte el orquestador en motor de procesamiento | Memoria del worker |
| Motor de procesamiento distribuido | Volumen que el warehouse no maneja bien, ML | Cluster que operar, otro skill set | Alto |

**Recomendación por defecto**: SQL en el warehouse. Salí de ahí solo cuando el problema no se pueda expresar en SQL, no cuando el SQL sea incómodo.

---

## Decisión: materialización de un modelo

| Opción | Conviene cuando | Contra |
|---|---|---|
| `view` | Staging, modelos poco consultados, lógica liviana | Recalcula en cada lectura |
| `table` | Marts chicos/medianos muy consultados | Reconstruye todo en cada corrida |
| `incremental` | Tablas grandes que crecen por append | Complejidad real: lookback, updates, divergencia |
| `ephemeral` | Paso intermedio sin consumo directo, pocos usos | No inspeccionable; infla el SQL de quien lo usa |
| Materialized view / dynamic table | Frescura casi continua sin orquestar | Consume compute de forma continua, haya o no consumidores |

**Recomendación por defecto**: vista en staging, tabla en marts. Pasar a incremental **solo** cuando el costo de reconstrucción lo justifique con números.

---

## Decisión: estrategia incremental

| Opción | Maneja updates | Conviene cuando | Contra |
|---|---|---|---|
| `append` | ❌ | Eventos inmutables puros | Duplica si se reprocesa |
| `merge` | ✅ | Default con clave única confiable | Costoso en tablas muy grandes |
| `delete+insert` | ✅ por ventana | Reemplazo de períodos completos | Ventana de inconsistencia si no es transaccional |
| `insert_overwrite` | ✅ por partición | Datos particionados por fecha | La partición debe ser la unidad natural |
| Full-refresh siempre | ✅ trivialmente | Tablas chicas | Costo de reconstrucción |

**Recomendación por defecto**: `merge` si hay clave única real; reemplazo por partición si el dato es naturalmente particionado por fecha. Y si la tabla es chica, full-refresh — es la opción correcta más veces de lo que la gente cree.

---

## Decisión: frecuencia del pipeline

| Opción | Latencia | Costo relativo | Conviene cuando |
|---|---|---|---|
| Diario | horas | 1x | Reporting, cierres, marts de análisis |
| Cada hora | ~1 hora | 3-5x | Dashboards operativos |
| Micro-batch (15 min) | minutos | 5-10x | Operación que revisa durante el día |
| CDC continuo | minutos | Medio + conector | Réplica de operacional sin castigar la fuente |
| Streaming | segundos | Alto + on-call | Acción automática con ventana corta |

**Recomendación por defecto**: la frecuencia se define por la decisión que se toma con el dato, no por la que se puede. La pregunta que ordena: *¿cuál es el costo de que este dato tenga N minutos de atraso?*

---

## Decisión: escalar Snowflake

| Síntoma | Acción | Lo que NO hay que hacer |
|---|---|---|
| Query lenta con spilling a disco | Scale up (warehouse mayor) | Agregar clusters |
| Queries encoladas, cada una rápida | Scale out (multi-cluster) | Agrandar el warehouse |
| Costo alto sin uso proporcional | Bajar auto-suspend | Achicar el warehouse |
| Un equipo afecta a otro | Separar warehouses | Subir el tamaño "para que alcance" |
| Escaneo del 100% de particiones | Arreglar el predicado | Poner clustering key de entrada |

---

## Decisión: capa de serving

| Opción | Latencia | Concurrencia | Conviene cuando |
|---|---|---|---|
| BI conectado al warehouse | Segundos | Baja-media | Análisis y reporting |
| API sobre el warehouse con caché | Cientos de ms a segundos | Media | Dashboards internos, integraciones |
| Base de servicio alimentada por pipeline | Milisegundos | Alta | Producto de cara al usuario |
| Reverse ETL | Minutos | — | Llevar datos a herramientas operativas |
| Semantic layer | — | — | La definición de métrica se está duplicando |
| Acceso directo desde la app al warehouse | ❌ | ❌ | Prácticamente nunca |

**Recomendación por defecto**: API con caché para consumo interno. Base de servicio solo cuando la latencia comprometida no se alcanza de otra forma — es un pipeline más que mantener.

---

## Decisión: orquestación

| Opción | Conviene cuando | Contra |
|---|---|---|
| Airflow autogestionado | Control total, ya tenés el skill de operarlo | Operás scheduler, workers, metadata DB |
| Airflow gestionado (Astronomer, MWAA, Composer) | Equipo chico, no querés operar | Costo, versiones al ritmo del proveedor |
| Scheduler del warehouse (tasks nativas) | Todo el trabajo es SQL adentro del warehouse | Sin dependencias complejas ni fuentes externas |
| Scheduler del propio dbt (dbt Cloud) | Proyecto centrado en dbt, sin ingesta compleja | Se queda corto cuando hay que coordinar fuera de dbt |
| Cron | ❌ Prototipos únicamente | Sin reintentos, sin dependencias, sin visibilidad |

**Recomendación por defecto**: si tu trabajo es principalmente SQL sobre el warehouse y no coordinás sistemas externos, no metas Airflow. Cada herramienta nueva es un costo operativo permanente.

---

## Decisión: paginación al consumir una API

| Opción | Consistente con escrituras | Conviene cuando |
|---|---|---|
| Cursor / keyset | ✅ | Default para ingesta |
| Offset / página | ❌ | Solo sobre datos congelados |
| Snapshot / punto en el tiempo | ✅ | Exportaciones grandes, si el proveedor lo ofrece |
| Sin paginar | — | Solo con máximo acotado y garantizado |

**Si la API solo ofrece offset y no podés cambiarla**: merge por clave única + ventana de solapamiento + reconciliación de conteos. Y un ticket al proveedor, para que sea una limitación documentada y no una deuda tuya invisible.

---

## Decisión: autenticación de un pipeline

| Opción | Seguridad | Operación | Conviene cuando |
|---|---|---|---|
| Identidad gestionada / federada | Alta | Mínima (sin secreto) | Default cuando el proveedor lo permite |
| OAuth 2.0 client credentials | Alta | Media (rotación) | Estándar máquina a máquina |
| API key en almacén de secretos | Media | Baja | Cuando el proveedor no ofrece otra cosa |
| API key en variable de entorno | Baja | Mínima | Solo desarrollo |
| Certificado cliente | Muy alta | Alta | Requisitos regulatorios |

**Recomendación por defecto**: identidad sin secreto donde se pueda. Todo lo demás es gestión de secretos, que es trabajo permanente.

---

## Decisión: estrategia de CI para datos

| Opción | Costo | Confianza | Conviene cuando |
|---|---|---|---|
| Solo modificados + descendientes, con deferral | Bajo | Alta | Default para cualquier proyecto real |
| Proyecto completo | Muy alto | Alta | Proyectos chicos únicamente |
| Solo compilar y lint | Mínimo | Baja | Puerta rápida, insuficiente sola |
| Sobre muestra reducida | Bajo | Media-baja | Complemento, nunca reemplazo |
| Staging con copia de producción | Medio | Muy alta | Cambios de alto riesgo, migraciones |

**Recomendación por defecto**: modificados + descendientes con deferral, en esquema efímero por rama. Es la técnica que hace viable el CI de datos.

---

## Decisión: formato de almacenamiento en el lake

| Opción | Conviene cuando | Contra |
|---|---|---|
| Tabla nativa del warehouse | Un solo motor, equipo chico, cero operación | Los datos viven adentro del proveedor |
| Iceberg | Estándar abierto, varios motores, portabilidad | Compactación y mantenimiento a tu cargo |
| Delta Lake | Ecosistema Spark/Databricks | Históricamente centrado en Spark |
| Parquet suelto | ❌ | Sin ACID, sin deletes, escrituras parciales visibles |

**Recomendación por defecto**: tabla nativa del warehouse salvo que tengas una razón concreta de multi-motor o de portabilidad. Elegir un formato abierto "por si acaso" es complejidad operativa sin beneficio.

---

## Decisión: modelado del mart

| Opción | Conviene cuando | Contra |
|---|---|---|
| Star schema (Kimball) | Default para consumo BI | Más modelos, duplicación en dimensiones |
| One Big Table | Un solo consumidor, warehouse columnar | Escala mal con muchos consumidores |
| Snowflake schema | Dimensiones enormes con jerarquías repetidas | Más joins, peor experiencia en BI |
| Data Vault | Trazabilidad y auditoría extremas | Complejidad alta; no es capa de consumo |

**Recomendación por defecto**: star schema con grano declarado. La OBT es tentadora y funciona hasta que aparece el segundo consumidor con otra pregunta.

---

## Decisión: SCD (dimensiones que cambian)

| Opción | Conserva historia | Conviene cuando | Contra |
|---|---|---|---|
| Type 1 (sobrescribe) | ❌ | Correcciones, typos, atributos sin valor histórico | Se pierde el pasado |
| Type 2 (versiona filas) | ✅ | El negocio necesita "cómo era en ese momento" | Más filas, joins con rango de fechas |
| Type 3 (columna anterior) | Parcial | Solo interesa el valor previo | Historia limitada a un salto |

**Recomendación por defecto**: Type 1 salvo que alguien pueda nombrar la pregunta de negocio que requiere el histórico. Type 2 sin caso de uso es complejidad regalada.

---

## Cómo el mentor usa estas tablas

1. Identificá la decisión real que está sobre la mesa (a veces el usuario pregunta por la herramienta cuando la decisión es otra).
2. Elegí UNA opción y decila con la razón. La tabla es respaldo, no la respuesta.
3. Nombrá el costo — en créditos, en tiempo o en carga operativa. En este dominio, una recomendación sin costo está incompleta.
4. Nombrá cuándo tu recomendación **dejaría** de valer. Eso es lo que la convierte en criterio y no en dogma.
