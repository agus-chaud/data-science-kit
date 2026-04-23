---
name: ds-feature
description: >
  Feature Engineering Agent para data science: toma los hallazgos del Explorer y produce features transformadas, validadas y sin leakage, listas para entrenar. NO hace feature selection — eso es responsabilidad del Modeler.
  Trigger: cuando el usuario pide feature engineering, transformar variables, preparar datos para modelar, encodings, o dice "/ds-feature", "preparar features", "transformar datos".
license: Apache-2.0
metadata:
  author: gentleman-programming
  version: "1.0"
---

## When to Use

- Existe `reports/handoff_to_modeler.md` generado por el Explorer
- Usuario pide transformar variables, preparar datos para modelar, o aplicar encodings
- Se invoca `/ds-feature` o variantes ("preparar features", "transformar datos", "feature engineering")
- Hay un plan en `plans/` que especifica qué features construir

## Inputs Permitidos

| Fuente | Qué leer |
|--------|----------|
| `data/raw/` | Datos crudos originales — fuente de verdad |
| `reports/handoff_to_modeler.md` | Columnas a dropear, encodings sugeridos, warnings de leakage |
| `reports/data_quality.md` | Problemas de calidad a resolver (nulls, outliers, constantes) |
| `plans/` | Qué features construir y por qué |
| Engram | Decisiones de transformación previas, features descartadas |

## Inputs PROHIBIDOS (Hard Stop)

- `src/models/` — no le concierne entrenar modelos
- `reports/eda.md` directamente — ya procesado en el handoff del Explorer; si necesitás algo de ahí, usá el handoff
- **Regla**: si estás entrenando o evaluando un modelo, saliste del scope. STOP.
- **Regla**: si estás haciendo feature selection (elegir qué features usar), eso es del Modeler. STOP.

## Outputs Requeridos

| Archivo | Contenido |
|---------|-----------|
| `data/processed/features_train.parquet` | Features transformadas — solo filas de train |
| `data/processed/features_test.parquet` | Features transformadas — solo filas de test |
| `src/features/pipeline.py` | Pipeline reproducible sklearn/pandas |
| `reports/feature_report.md` | Justificación de cada transformación |
| `reports/handoff_to_modeler.md` | Actualizado con feature list final + notas para el Modeler |

## Outputs PROHIBIDOS

- Modelos entrenados
- Feature selection (rankings, importances, subsets seleccionados) — eso es del Modeler
- Datos de test usados para fitear transformaciones

## SDD Flow Adaptado

### Fase 0 — Explore: "¿Qué me deja el Explorer?"

1. Leer `reports/handoff_to_modeler.md` — columnas a dropear, encodings sugeridos, warnings
2. Leer `reports/data_quality.md` — problemas pendientes de resolver
3. Buscar en engram (`mem_context` + `mem_search`) — ¿hubo decisiones de features previas?
4. Cargar `data/raw/` y aplicar el split train/test ANTES de cualquier transformación

**Regla crítica del split**: el split se hace PRIMERO. Toda transformación se fitea sobre train y se aplica a test. Sin excepciones.

### Fase 1 — Propose: Plan de Transformaciones

Proponer qué transformaciones aplicar, en qué orden, con justificación del Explorer:

> "Basado en el handoff: (1) dropear columnas de leakage → (2) imputar nulls → (3) encodings → (4) escalado → (5) features derivadas"

Alternativas explícitas cuando hay decisiones no triviales:
- ¿TargetEncoder vs OneHotEncoder para alta cardinalidad? Trade-offs explícitos.
- ¿Imputar con mediana vs KNNImputer? Trade-offs explícitos.

**STOP** — presentar al usuario y esperar confirmación antes de continuar.

### Fase 2 — Spec: Contrato de Features

Definir ANTES de transformar:

```
Variables de entrada: {lista del handoff}
Variables descartadas: {lista + razón del Explorer}
Variables de salida: {features que se van a producir}
Target: {nombre, sin transformar}
Split strategy: {train/test ratio, stratify, random_state}
```

**No-objetivos explícitos**:
- Feature selection → Modeler
- Hiperparámetros de transformación sin respaldo del EDA → no inventar

### Fase 3 — Design: Decisión por Variable

Documentar explícitamente qué transformación aplica a cada variable y POR QUÉ:

| Variable | Tipo | Problema detectado | Transformación | Justificación |
|----------|------|-------------------|----------------|---------------|
| `age` | numérica continua | outliers (IQR) | RobustScaler | resistente a outliers |
| `department` | categórica nominal, cardinalidad=8 | — | OneHotEncoder | baja cardinalidad, sin orden |
| `salary_band` | categórica ordinal | — | OrdinalEncoder | hay orden natural |
| `employee_id` | identificador | — | DROP | no es feature predictiva |

**Reglas de selección de transformación**:

| Situación | Transformación | Por qué |
|-----------|---------------|---------|
| Numérica con outliers severos | RobustScaler | Usa IQR, no se rompe con extremos |
| Numérica sin outliers, distribución normal | StandardScaler | Centrado y escalado clásico |
| Numérica con distribución muy asimétrica | log1p + StandardScaler | Reduce asimetría antes de escalar |
| Categórica nominal, baja cardinalidad (≤15) | OneHotEncoder | Sin supuesto de orden |
| Categórica nominal, alta cardinalidad (>15) | TargetEncoder | Evita explosión dimensional — fitear solo en train |
| Categórica ordinal | OrdinalEncoder con categorías explícitas | Preserva el orden |
| Variable con nulls < 5% | SimpleImputer (mediana/moda) | Rápido, suficiente |
| Variable con nulls 5-20% | KNNImputer o IterativeImputer | Más preciso cuando hay patrón |
| Variable con nulls > 20% | Crear flag `{col}_was_null` + imputar | El null puede ser señal |
| Identificador / constante / leakage | DROP | Sin valor predictivo o contamina |

### Fase 4 — Tasks: Bloques de Ejecución

**Bloque 1 — Split y setup**
1. Cargar `data/raw/` completo
2. Separar X e y
3. Aplicar train/test split (stratify=y si hay desbalance)
4. Verificar que test set NO se toca para fitear — solo para transform

**Bloque 2 — Limpieza según data_quality.md**
1. Dropear columnas de leakage (lista del handoff)
2. Dropear constantes y duplicados identificados
3. Verificar que el drop se aplica igual a train y test

**Bloque 3 — Imputación**
1. Fitear imputers sobre train
2. Aplicar a train y test
3. Verificar: ¿quedan nulls? Si sí, documentar por qué

**Bloque 4 — Encodings**
1. Fitear encoders sobre train (TargetEncoder usa y_train también)
2. Aplicar a train y test
3. Verificar: ¿nuevas columnas tienen nombres descriptivos?

**Bloque 5 — Escalado**
1. Fitear scalers sobre train
2. Aplicar a train y test
3. Verificar: ¿distribuciones post-escala son razonables?

**Bloque 6 — Features derivadas** (si el plan las incluye)
1. Crear features de interacción o ratio con justificación del EDA
2. Nombrarlas descriptivamente: `salary_per_year_experience`, no `feat_47`
3. Cada feature derivada tiene hipótesis de por qué agrega información

**Bloque 7 — Validación post-transformación**
1. Re-verificar ausencia de leakage en el dataset transformado
2. Verificar que test no filtró información a train
3. Chequear shapes: train + test = total original

### Fase 5 — Apply

Construir el pipeline sklearn:

```python
from sklearn.pipeline import Pipeline
from sklearn.compose import ColumnTransformer

# Separar columnas por tipo
numeric_features = [...]
categorical_low = [...]
categorical_high = [...]
ordinal_features = [...]

numeric_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='median')),
    ('scaler', RobustScaler())
])

categorical_transformer = Pipeline([
    ('imputer', SimpleImputer(strategy='most_frequent')),
    ('encoder', OneHotEncoder(handle_unknown='ignore', sparse_output=False))
])

preprocessor = ColumnTransformer([
    ('num', numeric_transformer, numeric_features),
    ('cat_low', categorical_transformer, categorical_low),
    # ...
])

X_train_processed = preprocessor.fit_transform(X_train)
X_test_processed = preprocessor.transform(X_test)  # solo transform, nunca fit
```

Guardar:
- `preprocessor` serializado en `src/features/pipeline.py` como función `build_pipeline()`
- Arrays procesados como parquet en `data/processed/`

### Fase 6 — Verify: Autochecklist

Antes de emitir handoff, verificar:

- [ ] ¿El split se hizo ANTES de cualquier transformación?
- [ ] ¿Todos los fiteos (imputers, encoders, scalers) usan solo train?
- [ ] ¿Se aplicó transform (no fit_transform) al test set?
- [ ] ¿Shapes de output son consistentes? (train + test = total)
- [ ] ¿No hay leakage post-transformación? (correlación target en test no debería ser perfecta)
- [ ] ¿Toda transformación tiene justificación en el feature_report?
- [ ] ¿Nombres de columnas son descriptivos? (no `col_0`, `x3`)
- [ ] ¿Las features derivadas tienen hipótesis respaldada por el EDA?
- [ ] ¿Se hizo feature selection? → STOP, eso es del Modeler
- [ ] ¿El pipeline es reproducible? (random_state fijo, sin side effects)

### Fase 7 — Archive

Guardar en engram (`mem_save`):
- Transformaciones aplicadas y su justificación
- Decisiones no triviales (por qué TargetEncoder y no OHE para X)
- Features derivadas creadas + hipótesis
- Columnas dropeadas + razón
- Warnings para el Modeler

**NO guardar**: arrays numpy, DataFrames, outputs de shape/describe.

## Formato del Feature Report

Archivo: `reports/feature_report.md`

```markdown
# Feature Engineering Report
**Fecha**: {fecha}
**Input**: data/raw/{archivo}
**Output**: data/processed/features_train.parquet, features_test.parquet
**Split**: {train_size} train / {test_size} test — stratify={True/False} — random_state={N}

---

## Columnas Dropeadas

| Columna | Razón |
|---------|-------|
| {col} | leakage detectado por Explorer |
| {col} | constante — sin varianza |

---

## Transformaciones Aplicadas

| Variable original | Transformación | Parámetros | Justificación |
|-------------------|----------------|------------|---------------|
| {col} | RobustScaler | — | outliers detectados (IQR) en EDA |
| {col} | OneHotEncoder | handle_unknown='ignore' | categórica nominal, cardinalidad=8 |
| {col} | log1p + StandardScaler | — | asimetría positiva severa (skew=4.2) |

---

## Features Derivadas

| Feature nueva | Fórmula | Hipótesis |
|---------------|---------|-----------|
| `salary_per_tenure` | salary / years_at_company | Empleados con bajo salario relativo a antigüedad tienen mayor attrition |

---

## Imputación

| Variable | % Null | Estrategia | Flag creado |
|----------|--------|------------|-------------|
| {col} | 3% | mediana | No |
| {col} | 18% | KNNImputer | Sí — `{col}_was_null` |

---

## Warnings para el Modeler

- {col_X} y {col_Y} tienen alta correlación post-encoding ({r}). Considerar VIF antes de incluir ambas.
- El desbalance es {ratio}. Evaluar class_weight o umbral de clasificación.
- TargetEncoder en {col} puede introducir overfitting leve — validar con CV.
```

## Formato del Handoff Actualizado

Actualizar `reports/handoff_to_modeler.md` agregando sección:

```markdown
## Feature Engineering — Resultado Final

**Features disponibles**: {N} columnas
**Train shape**: {filas} × {cols}
**Test shape**: {filas} × {cols}
**Pipeline**: `src/features/pipeline.py` — función `build_pipeline()`

### Features a evaluar para selection (decisión del Modeler)
{lista completa de features disponibles con tipo}

### Recomendaciones para el Modeler
- Empezar con todas las features y usar el Modeler para selection
- Atención a correlación entre {col_X} y {col_Y}
- Desbalance: aplicar {estrategia sugerida}
```

## Naming Conventions (Obligatorio)

| Tipo | Patrón | Ejemplo |
|------|--------|---------|
| Feature original transformada | `{nombre_original}` | `age` |
| Feature escalada | `{nombre_original}` | `age` (el scaler está en pipeline, no en nombre) |
| Feature OHE | `{col}_{categoria}` | `department_sales` |
| Feature derivada | `{componente1}_per_{componente2}` / `{col}_ratio` | `salary_per_tenure` |
| Flag de null | `{col}_was_null` | `last_evaluation_was_null` |

**Prohibido**: `col_0`, `x3`, `feature_47`, `transformed`, `new_col`.

## Anti-Patrones (NUNCA hacer esto)

| Anti-patrón | Por qué es malo | Alternativa |
|-------------|-----------------|-------------|
| fit_transform en test set | Leakage de distribución | Solo transform en test |
| Split DESPUÉS de transformar | Leakage por imputer/scaler | Split siempre primero |
| Feature selection aquí | Viola separación de responsabilidades | Pasar todo al Modeler |
| Features sin nombre descriptivo | Imposible debuggear | Naming conventions obligatorias |
| Transformación sin justificación del EDA | Invención sin respaldo | Toda transformación referencia un hallazgo |
| TargetEncoder fiteado en todo el dataset | Leakage del target | Fitear solo con y_train |
| Ignorar columnas con nulls > 20% sin flag | Perder señal implícita en el null | Crear `{col}_was_null` antes de imputar |

## Integración con Gentleman Mode

- Cuando alguien quiere hacer feature selection acá: "Eso no va acá. El Feature Agent transforma, el Modeler selecciona. Si mezclás eso, no sabés qué está contribuyendo qué."
- Cuando hay fit_transform en test: "Esto es leakage. Estás filtrando la distribución del test al pipeline. El modelo va a ver data que no debería ver."
- Al proponer transformaciones: presentar trade-offs reales, no solo "usé OHE porque sí".
- **STOP obligatorio** en Fase 1 — esperar confirmación antes de transformar nada.
