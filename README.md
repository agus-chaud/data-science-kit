#  Data Science Kit 

 Conjunto de skills especializadas que dan disciplina basada en roles a los proyectos de data science. Cada skill codifica las responsabilidades, restricciones y flujo de trabajo de una fase específica del proceso , previniendo los errores más comunes por diseño.

## El problema

Los proyectos de data science fallan de maneras predecibles:
- El EDA se mezcla con el feature engineering que se mezcla con el modelado , nadie sabe quién es responsable de qué
- El leakage se introduce sin que nadie lo detecte hasta producción
- Los modelos se eligen por intuición, no por un criterio escrito antes de ver los resultados
- El reporte final es un documento técnico que ningún ejecutivo entiende

Estas skills resuelven eso dándole a cada fase un **contrato estricto**: qué lee, qué escribe, qué tiene prohibido hacer, y cómo hace el handoff al siguiente agente.

## El ecosistema

```
ds-env-bootstrap --> ds-planner --> ds-dq (opc.) / ds-explorer --> ds-feature --> ds-model --> ds-reviewer --> ds-report
```

(`ds-dq` audita y corrige calidad con pandas según `calidad-de-datos.md`; puede ir antes o en paralelo al Explorer según el proyecto.)

---

## Skills

### `ds-env-bootstrap` — Entorno reproducible
Detecta dependencias probables a partir de la consigna del proyecto y arma un entorno Python reproducible: entorno virtual (`.venv`), `requirements.in` sin pin, lockfile `requirements.txt` congelado con versiones exactas, verificación mínima de imports y resumen en `reports/setup_env_report.md`. Prioriza que todos instalen desde el mismo lockfile. Incluye script opcional para inferir dependencias desde consigna (`.docx`, texto o markdown) cuando todavía no existe `requirements.in`.

**Invocar con**: preparar entorno, bootstrap de proyecto, setup de notebook, `requirements.txt`, `venv`, consigna de TP.

---

### `ds-planner` — Agente Planificador
Toma un objetivo ambiguo y lo parte en fases pequeñas y verificables con criterios de aceptación binarios.

**Inputs prohibidos**: `data/raw/`, `data/processed/`, `notebooks/` — el planner NO mira datos.
**Output**: `plans/PLAN_{fecha}_{tema}.md` con fases numeradas, cada una con objetivo, entregable, criterio de aceptación, estimación de esfuerzo, dependencias y agente sugerido.

**Invocar con**: `/ds-plan`, "planificar", "armar plan", "qué atacamos primero"

---

### `ds-explorer` — Agente Explorador
Convierte data cruda en comprensión — perfila, detecta problemas de calidad, genera y valida hipótesis de negocio.

**Inputs prohibidos**: `data/processed/`, `src/models/`
**Outputs**: `reports/eda.md`, `reports/hipotesis.md`, `reports/data_quality.md`, `notebooks/01_eda.ipynb`, `reports/handoff_to_modeler.md`

Toda hipótesis requiere: enunciado, test estadístico, resultado numérico, interpretación de negocio y recomendación. Las cuatro cosas — o no es una hipótesis.

**Invocar con**: `/ds-explore`, "explorá los datos", "qué hay en el dataset"

---

### `ds-dq` — Calidad de datos (pandas: diagnóstico y corrección)

Perfilado estructural, inventario de nulos/duplicados/tipos/cardinalidad, correlaciones **descriptivas**, y **corrección reproducible** (`astype`, `to_numeric`/`to_datetime`, `drop`/`dropna`/`fillna`, `drop_duplicates`, `rename`, `replace`/`map`, accessor `.str`). Contenido alineado con **`calidad-de-datos.md`** del kit (Pandas IV/V, caso Madrid, prácticas 13–15).

**No hace:** hipótesis de negocio con test+p-valor ni EDA ML completo (eso es `ds-explorer`); inferencia profunda (`ds-stats`); pipelines de features (`ds-feature`).

**Outputs típicos:** `reports/data_quality.md`, `reports/data_cleaning_log.md`, opcional `notebooks/00_calidad_datos.ipynb`.

**Invocar con**: `/ds-dq`, "calidad de datos", "limpiar el Excel", "diagnóstico de nulos y duplicados", "auditar calidad del dataset".

---

### `ds-stats` — Estadística (marco y rigor inferencial)
Orienta y explica estadística descriptiva e inferencial: elección e interpretación de tests, intervalos de confianza, α y p-valor, muestreo y tamaño de muestra, diseño y lectura de A/B, supuestos de modelos clásicos (normalidad, heterocedasticidad, linealidad, multicolinealidad) y trampas habituales (correlación vs causalidad, penetración vs distribución).  

**Límites operativos**:  
- No sustituye al `ds-explorer`: no arma `notebooks/01_eda.ipynb` ni el reporte EDA.  
- No sustituye al `ds-dq`: no es el playbook de limpieza/diagnóstico pandas del `calidad-de-datos.md`.  
- No sustituye al `ds-feature`: no implementa pipelines de transformaciones productivas.  
- No compite con `ds-model` en selección de variables o evaluación de modelos.  

**Rol transversal**: suele entrar después del EDA y durante iteraciones de modelado cuando hay que defender una decisión inferencial.

**Integración con Gentleman**:
- Separa significación estadística de tamaño de efecto e impacto de negocio.
- Explicita qué se puede concluir (y qué no) según el diseño real de los datos.

**Decision logging (cuando aplica)**:
- Si durante la tarea hubo decisiones inferenciales críticas (elección de test, una vs dos colas, α no estándar, supuestos aceptados/rechazados), `ds-stats` las puede registrar en `decisions.md` al cierre.

**Invocar con**: `/ds-stats`, "qué test uso", "interpretar IC", estadística, muestreo, A/B, p-valor, supuestos del modelo.

---

### `ds-feature` — Agente de Feature Engineering
Toma los hallazgos del Explorer y produce features transformadas, validadas y sin leakage, listas para entrenar. **No hace feature selection** , eso le corresponde al Modeler.

**Regla dura**: split PRIMERO, transformar DESPUÉS. Siempre. Todo encoder y scaler se fitea solo sobre train.
**Outputs**: `data/processed/features_train.parquet`, `data/processed/features_test.parquet`, `src/features/pipeline.py`, `reports/feature_report.md`

**Invocar con**: `/ds-feature`, "preparar features", "transformar datos"

---

### `ds-model` — Agente Modelador
Construye pipelines reproducibles, hace feature selection, entrena modelos, compara con rigor y elige el ganador con justificación cuantitativa.

**Reglas duras**:
- Baseline dummy obligatorio — sin baseline no hay comparación válida
- Mínimo 4 métricas: F1, Recall, Precision, PR-AUC
- Criterio del ganador escrito ANTES de ver los resultados — nada de cherry-picking
- Test set tocado exactamente UNA VEZ, al final

**Outputs**: `src/models/train.py`, `models/*.pkl`, `reports/modeling_results.md`, `notebooks/02_modelado.ipynb`

**Invocar con**: `/ds-model`, "entrenar", "modelar", "comparar modelos"

---

### `ds-reviewer` — Agente Revisor
QA crítico independiente — encuentra errores, bugs metodológicos y agujeros en el razonamiento. **No escribe código ni parches. Nunca.** Su poder viene de la independencia.

**Cada hallazgo requiere**: ubicación exacta (`archivo:línea/celda`), descripción, por qué es un problema, buena práctica violada (con nombre) y sugerencia de corrección.
**Escala de severidad**: BLOQUEANTE (invalida resultados) / ALTO / MEDIO / BAJO / POSITIVO
**Obligatorio**: mínimo 3 hallazgos positivos por revisión.

**Output**: `reports/review_{fecha}.md`

**Invocar con**: `/ds-review`, "revisá esto", "auditá el análisis"

---

### `ds-report` — Agente Escritor
Traduce hallazgos técnicos en un documento ejecutivo accionable para decisores no técnicos. **No hace análisis** , traduce análisis ya hechos.

**Inputs prohibidos**: `src/`, notebooks crudos, `data/`
**Estructura fija** (siempre en este orden): TL;DR → Problema de negocio → Qué encontramos → Cómo funciona el modelo → Recomendación accionable → Limitaciones → Próximos pasos
**Reglas duras**: cero jerga sin traducir, toda métrica con interpretación de negocio, toda recomendación con verbo + objeto + impacto esperado, 4-6 páginas máximo.

**Output**: `reports/executive_summary.md`, `reports/executive_summary.pdf`

**Invocar con**: `/ds-report`, "escribí el reporte", "resumí los resultados"

---

### `senior-ai-engineer-mentor` — Mentor de AI Engineering
Skill transversal, no forma parte del pipeline `ds-*`. Mentor activo (voz Senior/Solutions Architect) para aprender AI Engineering: fundamentos, RAG/MCP, APIs/microservicios, orquestación, multi-agente y producción. Explica conceptos, prepara para entrevistas, revisa agentes propios y planifica proyectos — no ejecuta tareas operativas.

**Invocar con**: "explicame", "qué es", "cómo funciona", "diferencia entre", "no entiendo", "preparame para entrevista", "revisá mi agente", `/ai-mentor`. Silenciar el turno con `/no-mentor`.

---

### `senior-data-engineer-mentor` — Mentor de Data Engineering y APIs
Skill transversal, no forma parte del pipeline `ds-*`. Mentor activo (voz Senior/Solutions Architect) para Snowflake, dbt, Airflow, Azure DevOps, diseño de sistemas back/front, APIs y MCP. Explica conceptos, prepara para entrevistas, revisa modelos dbt / DAGs propios y planifica pipelines — no ejecuta tareas operativas.

**Invocar con**: "explicame", "qué es", "cómo funciona", "por qué mi query es cara", "no entiendo", "preparame para entrevista", "revisá mi modelo dbt", "revisá este DAG", `/de-mentor`. Silenciar el turno con `/no-mentor`.

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/agus-chaud/data-science-kit.git
```

### 2. Copiar las skills al directorio de Claude Code

Todas las skills disponibles:

| Skill | macOS / Linux | Windows (PowerShell) |
|---|---|---|
| `ds-env-bootstrap` | `cp -r data-science-kit/skills/ds-env-bootstrap ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-env-bootstrap $env:USERPROFILE\.claude\skills\` |
| `ds-planner` | `cp -r data-science-kit/skills/ds-planner ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-planner $env:USERPROFILE\.claude\skills\` |
| `ds-explorer` | `cp -r data-science-kit/skills/ds-explorer ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-explorer $env:USERPROFILE\.claude\skills\` |
| `ds-dq` | `cp -r data-science-kit/skills/ds-dq ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-dq $env:USERPROFILE\.claude\skills\` |
| `ds-stats` | `cp -r data-science-kit/skills/ds-stats ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-stats $env:USERPROFILE\.claude\skills\` |
| `ds-feature` | `cp -r data-science-kit/skills/ds-feature ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-feature $env:USERPROFILE\.claude\skills\` |
| `ds-model` | `cp -r data-science-kit/skills/ds-model ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-model $env:USERPROFILE\.claude\skills\` |
| `ds-reviewer` | `cp -r data-science-kit/skills/ds-reviewer ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-reviewer $env:USERPROFILE\.claude\skills\` |
| `ds-report` | `cp -r data-science-kit/skills/ds-report ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\ds-report $env:USERPROFILE\.claude\skills\` |
| `senior-ai-engineer-mentor` | `cp -r data-science-kit/skills/senior-ai-engineer-mentor ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\senior-ai-engineer-mentor $env:USERPROFILE\.claude\skills\` |
| `senior-data-engineer-mentor` | `cp -r data-science-kit/skills/senior-data-engineer-mentor ~/.claude/skills/` | `Copy-Item -Recurse data-science-kit\skills\senior-data-engineer-mentor $env:USERPROFILE\.claude\skills\` |

Instalá solo la que necesites, copiando su línea. O instalá todas de una:

**macOS / Linux:**
```bash
cp -r data-science-kit/skills/ds-env-bootstrap ~/.claude/skills/
cp -r data-science-kit/skills/ds-planner ~/.claude/skills/
cp -r data-science-kit/skills/ds-explorer ~/.claude/skills/
cp -r data-science-kit/skills/ds-dq ~/.claude/skills/
cp -r data-science-kit/skills/ds-stats ~/.claude/skills/
cp -r data-science-kit/skills/ds-feature ~/.claude/skills/
cp -r data-science-kit/skills/ds-model ~/.claude/skills/
cp -r data-science-kit/skills/ds-reviewer ~/.claude/skills/
cp -r data-science-kit/skills/ds-report ~/.claude/skills/
cp -r data-science-kit/skills/senior-ai-engineer-mentor ~/.claude/skills/
cp -r data-science-kit/skills/senior-data-engineer-mentor ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
$skills = @("ds-env-bootstrap","ds-planner","ds-explorer","ds-dq","ds-stats","ds-feature","ds-model","ds-reviewer","ds-report","senior-ai-engineer-mentor","senior-data-engineer-mentor")
foreach ($s in $skills) {
    Copy-Item -Recurse "data-science-kit\skills\$s" "$env:USERPROFILE\.claude\skills\"
}
```

### 3. Verificar la instalación

Abrí Claude Code y ejecutá:
```
/ds-plan
```

Si Claude responde pidiendo el objetivo del proyecto, las skills están activas.

---

## Estructura recomendada del proyecto

```
tu-proyecto/
├── data/
│   ├── raw/          # solo lectura — ds-explorer lee acá
│   └── processed/    # ds-feature escribe acá
├── notebooks/
│   ├── 01_eda.ipynb
│   └── 02_modelado.ipynb
├── src/
│   ├── features/
│   │   └── pipeline.py
│   └── models/
│       └── train.py
├── models/           # modelos serializados
├── plans/            # ds-planner escribe acá
└── reports/          # todos los agentes escriben acá
    ├── eda.md
    ├── hipotesis.md
    ├── data_quality.md
    ├── handoff_to_modeler.md
    ├── feature_report.md
    ├── modeling_results.md
    ├── review_{fecha}.md
    └── executive_summary.md
```

---

## Licencia

Apache 2.0
