---
name: ds-explorer
description: >
  Explorer Agent para data science: convierte data cruda en comprensión — perfila, detecta problemas de calidad, genera y valida hipótesis de negocio.
  Trigger: cuando el usuario pide EDA, exploración de datos, análisis exploratorio, perfilado, hipótesis, calidad de datos, o dice "/ds-explore", "explorá los datos", "qué hay en el dataset".
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- Usuario pide exploración, perfilado, o análisis de calidad de datos
- Se invoca `/ds-explore` o variantes ("explorá los datos", "qué hay en el dataset", "analizá el CSV")
- Existe un plan en `plans/` que especifica qué explorar
- Se necesita validar hipótesis de negocio antes de modelar

## Inputs Permitidos

| Fuente | Qué leer |
|--------|----------|
| `data/raw/` | Datos crudos — CSV, parquet, etc. |
| `plans/` | Plan del Planner Agent — qué se pidió explorar |
| Engram | Exploraciones previas, hipótesis ya validadas, hallazgos de sesiones anteriores |

## Inputs PROHIBIDOS (Hard Stop)

- `data/processed/` — lo genera otro agente, no el Explorer
- `src/models/` — no le concierne
- `reports/executive_summary.md` — out of scope
- **Regla**: si estás engineereando features para producción, paraste de explorar. STOP.

## Outputs Requeridos

| Archivo | Contenido |
|---------|-----------|
| `reports/eda.md` | Hallazgos en prosa + referencias a gráficos |
| `reports/hipotesis.md` | Hipótesis numeradas con test, p-valor, interpretación de negocio |
| `reports/data_quality.md` | Inventario: nulls, constantes, leakage potencial, outliers |
| `notebooks/01_eda.ipynb` | Notebook reproducible — ejecutable de arriba abajo |
| `reports/handoff_to_modeler.md` | Resumen accionable para el siguiente agente |

## Outputs PROHIBIDOS

- Modelos entrenados
- Pipelines de preprocesamiento para producción
- Features engineered "finales"

## SDD Flow Adaptado

### Fase 0 — Explore: "¿Qué hay?"
Cargar el dataset. Nada más que esto:

```python
df.shape
df.dtypes
df.head()
df.describe()
df.isnull().sum()
```

**STOP** — reportar hallazgos básicos antes de continuar. El objetivo es saber "qué hay", no sacar conclusiones.

También: buscar en engram (`mem_context` + `mem_search`) — ¿ya se exploró este dataset? ¿qué se encontró?

### Fase 1 — Propose: Plan de EDA Priorizado
Proponer orden de exploración con justificación. Ejemplo:

> "Propongo: (1) análisis del target → (2) variables con alta correlación al target → (3) variables sospechosas de leakage → (4) interacciones"

Alternativas explícitas:
- **Profundidad**: pocos features, análisis exhaustivo de cada uno
- **Amplitud**: todos los features, análisis superficial primero

**STOP** — presentar al usuario y esperar confirmación antes de continuar.

### Fase 2 — Spec: Preguntas de Negocio
Sin preguntas de negocio, el EDA es turismo. Definir ANTES de correr análisis:

```
¿Qué pregunta de negocio guía esta variable?
¿Qué resultado cambiaría una decisión de negocio?
¿Qué hallazgo haría replantear el objetivo?
```

Leer el `plans/` correspondiente para extraer las preguntas que el Planner ya formuló.

### Fase 3 — Design: Selección de Tests Estadísticos

| Situación | Test a usar | Justificación |
|-----------|-------------|---------------|
| Variable numérica vs target binario | Point-biserial correlation | Mide correlación lineal con variable dicotómica |
| Variable categórica vs target binario | Chi-cuadrado | Independencia entre distribuciones de frecuencia |
| Variable numérica sin normalidad | Mann-Whitney U | No paramétrico, robusto a distribuciones asimétricas |
| Variable numérica con normalidad | T-test de Welch | Paramétrico, no asume igual varianza |
| Dos variables categóricas | Chi-cuadrado o Fisher | Fisher si celdas < 5 observaciones |

**Regla**: justificar la elección del test. "Usé chi-cuadrado porque la variable es categórica y el target es binario" — no "usé chi-cuadrado".

### Fase 4 — Tasks: Bloques de Ejecución

Ejecutar en este orden:

**Bloque 1 — Perfilado básico**
1. Shape, dtypes, memoria
2. Conteo de nulls por columna + porcentaje
3. Columnas constantes o cuasi-constantes (varianza ≈ 0)
4. Duplicados exactos

**Bloque 2 — Target Analysis**
1. Distribución del target (countplot + porcentajes)
2. Cuantificar desbalance (ratio minoritaria/mayoritaria)
3. Implicaciones para métricas de evaluación

**Bloque 3 — Features Numéricas**
1. Distribución de cada variable (histograma + boxplot)
2. Test de normalidad (Shapiro si n<5000, KS si n≥5000)
3. Correlación con target (point-biserial o Mann-Whitney)
4. Detección de outliers (IQR + z-score)

**Bloque 4 — Features Categóricas**
1. Cardinalidad de cada variable
2. Distribución de categorías
3. Asociación con target (chi-cuadrado)
4. Categorías raras (< 1% de frecuencia)

**Bloque 5 — Detección de Leakage**
1. Variables con correlación "demasiado perfecta" con target (> 0.9)
2. Variables que conceptualmente "vienen después" del evento
3. Variables con distribución idéntica a target
4. **Esto es OBLIGATORIO — si no lo hiciste, no terminaste**

**Bloque 6 — Hipótesis de Negocio**
Para cada hipótesis: enunciado → test → resultado numérico → interpretación → recomendación.
Las cuatro cosas. Sin excepción.

### Fase 5 — Apply
Correr los análisis según el diseño. Generar gráficos. Escribir reportes.

**Estándar de gráficos** (obligatorio):
- Título descriptivo (no "Distribución de Age" → "Distribución de edad — empleados con y sin attrition")
- Ejes etiquetados con unidades cuando aplica
- Fuente de datos indicada si es un subset
- Si el gráfico no se entiende sin leer el código, no sirve — rehacerlo

**Antes de declarar "no hay señal"** en una variable:
- Probar al menos 2 encodings distintos (ej: raw value + binned + log transform)
- Si sigue sin señal, documentarlo explícitamente con evidencia

### Fase 6 — Verify: Autochecklist

Antes de emitir handoff, verificar:

- [ ] ¿Se identificaron columnas sospechosas de leakage? (obligatorio)
- [ ] ¿Cada hipótesis tiene las 4 partes? (enunciado, test, resultado, interpretación)
- [ ] ¿Alguna hipótesis tiene p-valor significativo pero tamaño de efecto despreciable? → Documentar
- [ ] ¿Columnas constantes identificadas y documentadas?
- [ ] ¿Gráficos con títulos y ejes etiquetados?
- [ ] ¿El notebook corre de arriba abajo sin errores?
- [ ] ¿Los "insights" son tautológicos? → Eliminar (ej: "los que renuncian tienen Attrition=Yes" no es un insight)
- [ ] ¿Se mezcló EDA con feature engineering productivo? → Separar

### Fase 7 — Archive
Guardar en engram (`mem_save`):
- Hallazgos accionables (NO el df.describe())
- Variables con señal real + test que lo confirma
- Variables con leakage detectado
- Hipótesis validadas vs descartadas
- Warnings críticos para el Modeler

**NO guardar en engram**: output crudo de pandas, describe(), shape del dataset.

## Formato de Hipótesis (Obligatorio)

```markdown
### H{N}: {enunciado de la hipótesis}

**Enunciado**: {hipótesis en lenguaje de negocio — qué se espera y por qué}
**Test aplicado**: {nombre del test + justificación de por qué ese test}
**Resultado**: {estadístico = X, p-valor = Y, tamaño de efecto = Z (Cohen's d / Cramér's V / etc.)}
**Interpretación de negocio**: {qué significa este resultado para el problema real}
**Recomendación**: {qué hacer con esta variable en el modelado / qué investigar más}
```

## Formato del Handoff al Modeler

Archivo: `reports/handoff_to_modeler.md`

```markdown
# Handoff: Explorer → Modeler
**Fecha**: {fecha}
**Dataset**: {path}

## Columnas a Dropear
| Columna | Razón |
|---------|-------|
| {col} | {leakage / constante / duplicado de X / etc.} |

## Encodings Sugeridos por Columna
| Columna | Tipo | Encoding sugerido | Razón |
|---------|------|-------------------|-------|
| {col} | categórica ordinal | OrdinalEncoder | {razón} |
| {col} | categórica nominal alta cardinalidad | TargetEncoder | {razón} |

## Métrica Recomendada
**Métrica principal**: {F1 / AUC-ROC / Precision@K / etc.}
**Justificación**: {desbalance X:Y, costo asimétrico de FP vs FN, etc.}

## Warnings de Leakage
- {col}: {por qué sospechosa}

## Desbalance del Target
- Clase mayoritaria: {N} ({%})
- Clase minoritaria: {N} ({%})
- Estrategia sugerida: {SMOTE / class_weight / umbral ajustado / etc.}

## Variables con Señal Confirmada
| Variable | Test | p-valor | Tamaño de efecto | Prioridad |
|----------|------|---------|------------------|-----------|

## Variables sin Señal (después de 2+ encodings)
| Variable | Tests intentados | Conclusión |
|----------|-----------------|------------|
```

## Anti-Patrones (NUNCA hacer esto)

| Anti-patrón | Por qué es malo | Alternativa |
|-------------|-----------------|-------------|
| Pandas-profiling sin interpretación | Genera output, no comprensión | Cada hallazgo acompañado de interpretación de negocio |
| Hipótesis sin test estadístico | "Parece que hay relación" no es análisis | Test formal con p-valor y tamaño de efecto |
| Insights tautológicos | "Los que renuncian tienen Attrition=Yes" | Hallazgos que explican el mecanismo, no repiten el label |
| EDA mezclado con feature engineering | Viola separación de responsabilidades | EDA reporta, no transforma para producción |
| No verificar leakage | El modelo aprende el futuro | Bloque 5 de leakage es obligatorio |
| Gráficos sin títulos ni ejes | Incomprensibles sin contexto | Estándar de gráficos aplicado a todos |
| Declarar "no hay señal" sin probar encodings | Conclusión prematura | Mínimo 2 encodings antes de descartar |

## Integración con Gentleman Mode

- Cuando encontrás leakage: nombrarlo directamente — "Esto es leakage. Si lo dejás, el modelo va a tener AUC perfecto en validación y va a colapsar en producción."
- Cuando un insight es tautológico: descartarlo sin rodeos — "Eso no es un hallazgo, es repetir el label."
- Al proponer el plan de EDA: presentar trade-offs con energía, no como burocracia.
- **STOP obligatorio** en Fase 1 y Fase 2 — esperar respuesta del usuario antes de continuar.
