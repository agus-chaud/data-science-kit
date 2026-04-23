# 🤖 Data Science Kit — Skills para Claude Code

Un conjunto de skills especializadas para [Claude Code](https://claude.ai/code) que le dan **disciplina basada en roles** a los proyectos de data science. Cada skill codifica las responsabilidades, restricciones y flujo de trabajo de una fase específica del proceso — previniendo los errores más comunes por diseño.

## El problema

Los proyectos de data science fallan de maneras predecibles:
- El EDA se mezcla con el feature engineering que se mezcla con el modelado — nadie sabe quién es responsable de qué
- El leakage se introduce sin que nadie lo detecte hasta producción
- Los modelos se eligen por intuición, no por un criterio escrito antes de ver los resultados
- El reporte final es un documento técnico que ningún ejecutivo va a leer

Estas skills resuelven eso dándole a cada fase un **contrato estricto**: qué lee, qué escribe, qué tiene prohibido hacer, y cómo hace el handoff al siguiente agente.

## El ecosistema

```
ds-planner → ds-explorer → ds-feature → ds-model → ds-reviewer → ds-report
```

---

## Skills

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

### `ds-feature` — Agente de Feature Engineering
Toma los hallazgos del Explorer y produce features transformadas, validadas y sin leakage, listas para entrenar. **No hace feature selection** — eso le corresponde al Modeler.

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
Traduce hallazgos técnicos en un documento ejecutivo accionable para decisores no técnicos. **No hace análisis** — traduce análisis ya hechos.

**Inputs prohibidos**: `src/`, notebooks crudos, `data/`
**Estructura fija** (siempre en este orden): TL;DR → Problema de negocio → Qué encontramos → Cómo funciona el modelo → Recomendación accionable → Limitaciones → Próximos pasos
**Reglas duras**: cero jerga sin traducir, toda métrica con interpretación de negocio, toda recomendación con verbo + objeto + impacto esperado, 4-6 páginas máximo.

**Output**: `reports/executive_summary.md`, `reports/executive_summary.pdf`

**Invocar con**: `/ds-report`, "escribí el reporte", "resumí los resultados"

---

## Instalación

### 1. Clonar el repositorio

```bash
git clone https://github.com/agus-chaud/data-science-kit.git
```

### 2. Copiar las skills al directorio de Claude Code

**macOS / Linux:**
```bash
cp -r data-science-kit/skills/ds-planner ~/.claude/skills/
cp -r data-science-kit/skills/ds-explorer ~/.claude/skills/
cp -r data-science-kit/skills/ds-feature ~/.claude/skills/
cp -r data-science-kit/skills/ds-model ~/.claude/skills/
cp -r data-science-kit/skills/ds-reviewer ~/.claude/skills/
cp -r data-science-kit/skills/ds-report ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
$skills = @("ds-planner","ds-explorer","ds-feature","ds-model","ds-reviewer","ds-report")
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
