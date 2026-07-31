# Chuleta — Hito 5: APIs & MCP

> Referencia rápida. Para aprender de cero, andá a `milestones/05-apis-mcp.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `rest-resource-design` | Recursos + métodos estándar + status codes | "El código de estado es la interfaz con toda la infraestructura intermedia, no solo con el dev." |
| `api-contracts` | OpenAPI como acuerdo entre equipos | "Agregar no rompe; sacar o cambiar el tipo sí. Y el peor cambio es el que no se ve." |
| `api-pagination-filtering` | Cursor vs offset | "Offset es correcto solo sobre datos congelados." |
| `api-auth` | OAuth2, service principals, identidades | "El secreto no se guarda, se resuelve en ejecución. Mejor todavía: que no haya secreto." |
| `api-reliability` | Timeouts, reintentos, rate limits, idempotencia | "Toda llamada de red falla. La pregunta es qué hacés cuando falla." |
| `mcp-protocol` | Tools / resources / prompts | "Tools son acción, resources son contexto. Y el protocolo no te da seguridad: las políticas las ponés vos." |

## Tradeoff principal del hito — qué reintentar

| Situación | Acción correcta | Error común |
|---|---|---|
| Timeout de conexión | Reintentar con retroceso | No poner timeout |
| 429 (rate limit) | Esperar lo que indica el servidor | Reintentar inmediato |
| 5xx | Reintentar con retroceso + jitter | Bucle cerrado sin espera |
| 4xx (salvo 429) | No reintentar; fallar y alertar | Reintentar y quemar cuota |
| Escritura con timeout | Reintentar con clave de idempotencia | Reintentar sin clave y duplicar |

## Top 3 anti-patterns (con el fix en 1 línea)

1. Llamada de red sin timeout → la tarea queda colgada sin fallar ni alertar. Timeout siempre, más uno a nivel de tarea.
2. Paginación por offset sobre datos vivos → cursor; si no se puede, merge por clave + solapamiento + reconciliación.
3. Tool MCP que ejecuta SQL arbitrario → tools parametrizadas, identidad de solo lectura, cuotas y auditoría.

## La pregunta de entrevista que más cae

**Q:** La ingesta trae datos duplicados a veces. ¿Qué hacés?
**A (esqueleto):**
- No poner un `DISTINCT`. Eso tapa el síntoma y esconde el salteo, que es peor.
- Revisar la paginación: offset sobre una fuente con escrituras duplica y saltea.
- Revisar si hay reintentos sobre escrituras sin clave de idempotencia.
- Revisar si la escritura al destino es merge o `INSERT`.
- Reconciliar conteos contra la fuente: la duplicación se ve, el salteo hay que buscarlo.

## Decisión rápida (cheat)

- **¿Este cambio de API rompe?** Agregar opcional: no. Sacar, cambiar tipo, volver obligatorio: sí. Cambiar el significado sin cambiar el tipo: sí, y es el peor.
- **¿Cursor u offset?** ¿La fuente recibe escrituras mientras leés? → cursor obligatorio.
- **¿El orden del cursor es estable?** Tiene que ser total: fecha MÁS un desempate único.
- **¿Reintento esto?** ¿El error puede cambiar solo con el tiempo? Sí → reintento con retroceso y jitter. No → falla rápido.
- **¿Tool o resource en MCP?** ¿Tiene efecto o cuesta plata? → tool, con límites. ¿Es contexto de solo lectura? → resource.
- **¿Dónde va el secreto?** En el almacén, resuelto en ejecución. Si el proveedor permite identidad federada, en ningún lado.
