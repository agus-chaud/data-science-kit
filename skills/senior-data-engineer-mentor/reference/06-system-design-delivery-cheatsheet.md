# Chuleta — Hito 6: System Design & Delivery

> Referencia rápida. Para aprender de cero, andá a `milestones/06-system-design-delivery.md`. Esto es para repasar en 2 minutos.

## Los conceptos en 1 línea

| Concepto | Qué es | El one-liner senior |
|---|---|---|
| `backend-frontend-split` | Quién hace qué, y el patrón BFF | "El backend siempre autoriza. Lo del front es experiencia de usuario, no control." |
| `data-serving-layer` | Cómo sale el dato del warehouse | "El warehouse es un motor de cómputo, no un servidor de aplicaciones." |
| `azure-pipelines` | Stages/jobs/steps, templates, environments | "El pipeline es código de infraestructura. La plantilla obligatoria es donde vive el gobierno." |
| `cicd-for-data` | Validar y promover cambios de modelos | "El CI de datos no puede copiar al de software: acá cada verificación construye tablas de verdad." |
| `iac-secrets` | Key Vault, identidades, IaC | "Si un secreto entró al repo, ya está comprometido. Borrarlo no lo desactiva: rotarlo sí." |
| `data-governance-cost` | RBAC, lineage, contratos, SLO, atribución | "El equipo de datos tiene que enterarse antes que el negocio. Siempre." |

## Tradeoff principal del hito — capa de serving

| Patrón | Latencia | Concurrencia | Cuándo |
|---|---|---|---|
| BI sobre el warehouse | Segundos | Baja-media | Análisis y reporting |
| API sobre el warehouse + caché | Cientos de ms a segundos | Media | Dashboards internos, integraciones |
| Base de servicio | Milisegundos | Alta | Producto de cara al usuario |
| Reverse ETL | Minutos | — | Llevar el dato a herramientas operativas |
| App → warehouse directo | ❌ | ❌ | Prácticamente nunca |

## Top 3 anti-patterns (con el fix en 1 línea)

1. El front consultando el warehouse → API delgada con métricas agregadas, caché y autorización en el backend.
2. CI que construye el proyecto entero → modificados + descendientes con deferral, en esquema efímero por rama.
3. Nadie sabe quién consume qué → lineage para lo interno, log de consultas del warehouse para lo externo.

## La pregunta de entrevista que más cae

**Q:** ¿Cómo llevás un cambio de modelo a producción?
**A (esqueleto):**
- Cada desarrollador escribe en su propio esquema; nadie pisa a nadie.
- En el PR: construir solo lo modificado y sus descendientes en un esquema efímero de la rama, correr los tests, destruir el esquema al terminar.
- Análisis de impacto: lineage para los consumidores internos MÁS el log de consultas del warehouse para los externos que el lineage no ve.
- Cambios de alto riesgo (renombre, cambio de grano): paso extra con aprobación informada y plan de reversión.
- Al mergear, el despliegue actualiza producción; el job programado es el único que escribe ahí.

## Decisión rápida (cheat)

- **¿El front puede consultar el warehouse?** No. Cuatro razones: credenciales, autorización por fila, latencia y costo por usuario.
- **¿Qué capa de serving?** Empezá por latencia requerida y concurrencia esperada. Si tres áreas definen la métrica distinto, el problema es de propiedad, no de capa.
- **¿Variable vacía en el pipeline?** Casi siempre es un problema de momento de evaluación, no de valor.
- **¿Cómo hago barato el CI?** Construir solo lo modificado y sus descendientes, diferir el resto a producción.
- **¿El CI nunca falla?** Sospechá. Datos no representativos, tests vacíos, o exclusión de los modelos pesados.
- **¿Borré el secreto del commit?** No alcanza. Rotalo.
- **¿Quién se rompe si cambio esto?** El lineage solo ve lo interno. Los externos están en el log de consultas.
