# Calidad de datos con pandas (diagnóstico y corrección)

Notas sintetizadas del máster: **Pandas IV — Diagnóstico** (cheatsheet PDF, notebooks `10_…` y `11_…` ejercicios) y **Pandas V — Corrección** (cheatsheet `Cheatsheet_Pandas_Calidad_Datos_Correccion.pdf`, notebook `13_Pandas_Base_V_Calidad_de_Datos_Correccion.ipynb`, más la práctica **`14_…Ejercicios`** / **`15_…Soluciones`** sobre Madrid 2020). Primero conviene **caracterizar**; después **intervenir** de forma reproducible y alineada con negocio.

---

## 1. Por qué esta fase importa

En proyectos reales los datos **nunca** llegan perfectos. Conviene **invertir tiempo en diagnóstico** para listar huecos de calidad (nulos, duplicados, tipos equivocados, dominancia de categorías, combinaciones repetidas, etc.) **antes** de las fases siguientes. Sin ese mapa, arriesgamos decisiones equivocadas downstream.

---

## 2. Muestras (`sample`)

- `pandas.DataFrame.sample` sirve para subconjuntos (pruebas, exploraciones posteriores).
- Parámetros clave: `n` (tamaño absoluto), `frac` (fracción), `replace` (muestreo con/sin reposición), `random_state` (reproducibilidad).

**Recomendación del material:** en la fase de **calidad de datos** normalmente **no** conviene trabajar solo con una muestra, porque pueden quedar fuera errores sistemáticos. Conocer `sample()` sí es útil para **fases posteriores**.

```python
muestra = df.sample(n=100)
muestra = df.sample(frac=0.02, random_state=42)
```

---

## 3. Visión global del dataset

| Objetivo | Herramienta | Notas |
|----------|-------------|--------|
| Filas × columnas, tipos por columna, conteos no nulos | `df.info()` | Con `memory_usage='deep'` el uso de memoria es **real**, no solo estimado. |
| Dimensiones | `df.shape` | Tupla `(filas, columnas)`. |
| Índice | `df.index` | Tipo y longitud; `df.index.values` como array NumPy. |
| Nombres de columnas | `df.columns` | `df.columns.values` (array); `df.columns.to_list()` (lista). |
| Tipos pandas | `df.dtypes` | Vista global; `df["col"].dtype` para una columna. |
| Resumen numérico (y opcionalmente todo) | `df.describe()` | Por defecto **solo numéricas**; transpuesta `.T` mejora lectura. |

`describe` — parámetro **`include`**:

- Sin especificar: numéricas.
- `include='all'`: numéricas, objetos, bool, etc. (según soporte).
- `include=['O']`: orientado a **object** / categóricas típicas: `count`, `unique`, `top`, `freq`.

Eso permite detectar a simple vista categorías dominantes, cardinalidad alta o valores “raros” si se combinan con el negocio.

---

## 4. Identificación de nulos

- **Conteo por variable:** `df.isna().sum().sort_values(ascending=False)`
- **Proporción / porcentaje:** `df.isna().mean() * 100` (mismo orden sugerido con `sort_values`).

Interpretación: sirve para priorizar columnas para imputación, exclusion o tratamiento especial, y para alinear umbrales con el negocio (p. ej. “más del 20% de faltantes”).

---

## 5. Identificación de duplicados

- **Cuántos filas están duplicadas (fila completa):** `df.duplicated().sum()`
- **Ver las filas duplicadas:** `df[df.duplicated()]`

Parámetros importantes de `duplicated()`:

- **`subset`:** lista de columnas que definen duplicidad (la misma combinación repetida cuenta como duplicado).
- **`keep`:** `'first'` (marca desde el segundo igual), `'last'`, o **`False`** (marca **todas** las filas que pertenecen a un grupo duplicado, no solo la “segunda copia”).

Con `keep=False` puedes **listar todas las repeticiones** de cada caso duplicado (útil para inspección y para decidir si son errores de carga o registros legítimos repetidos):

```python
df[df.duplicated(keep=False)]
```

Los duplicados “reales” dependen del **criterio de negocio** (clave natural vs. combinación arbitraria).

---

## 6. Cardinalidad y valores distintos

- **`df.nunique()`:** número de valores únicos por columna. Parámetro `dropna`: si es `False`, también cuenta `NaN` como categoría única donde aplique.
- **`series.unique()`:** array de valores distintos (ojo con volumen si la cardinalidad es enorme).

Ejemplo práctico del material: alta `unique` en `Name` o `Use` sugiere textos muy diversos — útil junto con `value_counts` para ver si hay placeholders (“Anonymous”) o concentraciones.

---

## 7. Estadísticos básicos por tipo de variable

### Categóricas (u object)

- `value_counts()` — frecuencias; `normalize=True` proportiones entre 0 y 1 (×100 para %).
- `mode()` — moda(s); puede haber más de un valor modal.

### Continuas (numéricas)

- `mean()`, `median()`, `min()`, `max()`
- **`idxmax()` / `idxmin()`:** índice de la fila donde ocurre máximo/mínimo — útil para **inspeccionar el registro completo** (`df.loc[idx]`) ante outliers o casos límite.

### Correlación

Documentación: `df.corr` / `Series.corr`.

- **`method='pearson'` (por defecto):** relación **lineal** entre variables continuas; el propio material recuerda que es paramétricamente más defendible con **distribución aproximadamente normal** en ambas variables.
- **`method='kendall'` o `'spearman'`:** relaciones **monotónicas**; adecuadas para **ordinales** o continuas **no normales**. Ante la duda en el material se sugiere tender a **Kendall**.

Para toda la matriz numérica: `df.corr(numeric_only=True)` (evita mezclar tipos no numéricos).

### Filtrar por tipología antes de operar en bloque

`df.select_dtypes(...)` reduce el DataFrame a columnas de ciertos dtypes (p. ej. `'number'`, `'object'`, `bool`, o listas de tipos). Encaja con aplicar `mean()` solo a numéricas o `mode()` solo a object en un paso.

```python
df.select_dtypes("number").mean()
df.select_dtypes("object").mode()
```

---

## 8. Patrones de ejercicios (checklist reutilizable)

1. **Top-k columnas por % de nulos:** `df.isna().mean().sort_values(ascending=False).head(k).index.to_list()`
2. **Columnas por encima de umbral de faltantes:** `df.columns[df.isna().mean() > 0.2].to_list()`
3. **Categorías con baja representación:** `value_counts(normalize=True).sort_values().head(k) * 100`
4. **Duplicados por clave de negocio:** `df.duplicated(subset=[...]).sum()`
5. **Subconjunto por dimensión + estadístico:** p. ej. `df[df.Sector == "Retail"].select_dtypes("number").mean()`
6. **Registro asociado a un extremo:** `idx = df["Loan Amount"].idxmax(); df.loc[idx, "Country"]`
7. **Screening de campos mal codificados en categóricas:** `df.describe(include=["object"]).T` — revisar `unique`, `top`, `freq`.
8. **Duplicados: ver todas las filas involucradas:** `df[df.duplicated(keep=False)]`.
9. **Tras fijar un índice de negocio** (p. ej. expediente), **cuántas filas duplicadas hay por cada clave** en ese subconjunto: `df[df.duplicated(keep=False)].index.value_counts()`.

---

## 9. Referencia rápida (cheatsheet)

- Información general: `df.info()` — memoria profunda: `df.info(memory_usage="deep")`
- Dimensión: `df.shape`
- Variables: `df.columns` — tipos: `df.dtypes`
- Estadísticos: `df.describe().T`
- Nulos (conteo): `df.isna().sum().sort_values(ascending=False)`
- Nulos (%): `df.isna().mean().sort_values(ascending=False) * 100`
- Duplicados (conteo): `df.duplicated().sum()` — ver filas: `df[df.duplicated()]`
- Valores únicos (conteo por columna): `df.nunique()` — lista de valores: `df["Var1"].unique()`
- Frecuencias: `df["Var1"].value_counts()` — proporciones: `df["Var1"].value_counts(normalize=True) * 100`
- Moda: `df["Var1"].mode()`
- Media / mediana / min / max: `mean()`, `median()`, `min()`, `max()`
- Índice de extremos: `idxmax()`, `idxmin()`
- Correlación: `df["Var1"].corr(df["Var2"])` — ordinales / monotónica: `..., method="kendall"`

*(En la cheatsheet original aparece `pd.shape`; en un DataFrame el patrón correcto es `df.shape`.)*

---

## 10. Caso práctico (notebook 11): accidentalidad Madrid 2020

La práctica usa **Excel** (`2020_Accidentalidad.xlsx`), accidentes de tráfico en Madrid en 2020. El guion del curso insiste en un hábito sano: **ir anotando** todo lo raro o inconsistente en el diagnóstico para **corregirlo después** (otro módulo), sin mezclar aún limpieza con exploración.

### Carga y estructura

```python
df = pd.read_excel("ruta/2020_Accidentalidad.xlsx")
```

- Dimensiones del caso: **28 773 filas × 15 columnas**; memoria del orden de **~20 MB** con `info(memory_usage="deep")`.
- **Índice inicial:** `RangeIndex` automático (0 … n−1). Si el negocio identifica filas por expediente, tiene sentido **`df.set_index("Nº  EXPEDIENTE", inplace=True)`** (ojo con los espacios en el nombre literal de columna del Excel).

### Red flags típicos de origen Excel

- Columnas **`Unnamed: …`**: suelen ser celdas vacías o artefactos al exportar. En este dataset, **`Unnamed: 13`** puede llegar **toda nula** y **`Unnamed: 14`** casi solo nulos salvo **un valor** espurio (en el ejemplo, una coma `","`) — diagnóstico: candidatas a **eliminar** o a investigar en el fichero fuente.
- **`HORA` como `object`:** conviene revisar si conviene parsearla a tiempo (`datetime`/timedelta/`time`) para análisis temporales coherentes.

### Vista de categóricas (`describe(include="O")`)

La práctica fuerza **`df.describe(include="O").T`** cuando predominan objetos: se ve **cardinalidad** (`unique`), **modo** (`top`, `freq`) y **huecos implícitos** en `count` si filas tienen NA en esa columna.

Ejemplos interpretables del caso (orientativos):

- **`NÚMERO`**: muchas filas con `"-"` como valor modal — puede ser “sin número” en la fuente, no un dato perdido típico.
- **`RANGO DE EDAD`**: categoría **DESCONOCIDA** con frecuencia notable — problema de captura vs. código de falta valor.
- **`CALLE`** con alta cardinalidad (~miles): la calle con **más accidentes** coincide con la **moda** (`df["CALLE"].mode()`).

### Duplicados a nivel fila completa

- **`df.duplicated().sum()`** cuenta filas marcadas como duplicadas (por defecto la primera ocurrencia del grupo **no** cuenta). En este ejercicio salen **1 313** filas así.
- Para **audit visual** de todos los involucrados: **`df[df.duplicated(keep=False)]`**.
- Para **cuántas filas repetidas tiene cada expediente** (índice = expediente):  
  **`df[df.duplicated(keep=False)].index.value_counts()`** — ordena expedientes por severidad de repetición (p. ej. hasta 18 filas iguales para un mismo expediente en los outputs del notebook). Aquí puede debatirse si son **personas/implicados** en el mismo expediente (legítimo) o **copias errores**.

### Análisis de nulos

`df.isna().sum().sort_values(ascending=False)` ordena prioridades: en el caso, destacan **`LESIVIDAD*`** (~13k nulos), **`SEXO`**, **`ESTADO METEREOLÓGICO`**, y las columnas `Unnamed` casi todo vacío — coherente con tratamiento posterior (imputación, recodificación o exclusión).

### Variable numérica `LESIVIDAD*`

El ejercicio pide máximo, mínimo, media y mediana sobre la serie completa (pandas ignora NaN en agregaciones por defecto). En el ejemplo: **mediana > media**, señal de **asimetría** hacia valores bajos o colas; conviene complementar con histogramas/boxplots en otro paso del proyecto.

---

## 11. Corrección de datos (Pandas V)

Objetivos del notebook 13 (después del diagnóstico): **convertir tipos**, **eliminar o insertar variables/registros**, **tratar nulos**, **quitar duplicados**, **renombrar**, **reemplazar o recodificar valores**, **normalizar texto** con el accessor `.str`.

Regla práctica del material: **`astype`** es el camino genérico; **`to_numeric` / `to_datetime`** suelen comportarse mejor con datos “sucios”. Casi todas las operaciones vistas son **no destructivas por defecto** salvo donde se indica (`insert` sí modifica el DataFrame; el resto usa `inplace=True` solo si lo pides).

### 11.1 Conversión de tipos

#### `astype`

- Una columna: `serie.astype("category")`.
- Varias a la vez: `df.astype({"col1": "float", "col2": "category"})`.
- Por bloques: `df.select_dtypes("object").astype("category")` — en el ejercicio del notebook reduce memoria (**~2,4 MB → ~1,0 MB** en Kiva con `info(memory_usage="deep")`).
- **`astype` no modifica inplace** si no asignás el resultado (`df = df.astype(...)` o columna a columna).

#### Numéricos: `pd.to_numeric`

- Parámetro clave: **`errors`**; con **`errors="coerce"`** lo no convertible pasa a **NaN** (útil tras diagnóstico de strings mezclados con números).

#### Fechas: `pd.to_datetime`

- **`dayfirst=True`** si el primer campo es día (formato europeo).
- **`errors="coerce"`** para forzar NaN ante texto inválido.
- **`format="%d%m%Y"`** cuando el parseo ambiguo falla (ej. `'13082020'`).
- Códigos de formato: [strftime/strptime](https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior).
- Si ya es `datetime`, **solo presentación** (sale `object`): `serie.dt.strftime("%d/%m/%Y")`.

#### Categórica **ordinal**: `CategoricalDtype` y `.cat`

1. Definir orden: `orden = pd.CategoricalDtype(["a", "b", "c"], ordered=True)`.
2. Asignar: `df["col"] = df["col"].astype(orden)`.
3. Para **reordenar** categorías existentes: **`Series.cat.set_categories([...], ordered=True)`** — en producción suele asignarse de vuelta a la columna para persistir el cambio.

### 11.2 Eliminar o insertar columnas/filas

- **`drop`**: por nombre → `columns=[...]` o `index=[...]`. Por defecto **no** es `inplace`.
- **Eliminar filas por posición**: obtener etiquetas del índice (`df.index[0:5]`) y pasarlas a `drop(index=...)`.
- **`insert(loc, column, value)`**: inserta en posición entera; **siempre inplace**; **falla** si el nombre de columna ya existe.

### 11.3 Nulos: eliminar o rellenar

#### `dropna`

- **`axis=0`**: filas; **`axis=1`**: columnas.
- **`how="any"`** (default) vs **`how="all"`** (solo si toda la fila/columna es NA).
- **`subset`**: limitar el criterio a unas columnas.
- **`thresh`**: umbral de no-nulos para conservar fila/columna.

#### `fillna`

- **`value`**: escalar, o estadístico calculado (ej. moda: `s.mode().iloc[0]` o `.values[0]`).
- **Encadenado**: `ffill` / `bfill` como métodos (`serie.ffill()`, `serie.bfill()`) o dentro de `fillna` según versión de pandas.
- **`value_counts(dropna=False)`** para ver nulos como categoría explícita antes/después.

**Categóricas:** si imputás con un **valor nuevo** que **no** está en las categorías, antes **`serie.cat.add_categories("Nuevo")`** (u otro API de categorías) y luego `fillna`, según el flujo del curso.

**Nota de negocio:** imputar una **fecha** con la **moda** (ejercicio del notebook) es un patrón **solo mecánico**; en análisis temporal o ML puede **distorsionar** series o introducir sesgo — conviene documentar el supuesto.

### 11.4 Duplicados: eliminación

- **`drop_duplicates()`**: quita filas duplicadas; **no usa el índice** en la comparación (solo columnas).
- **`subset`**: columnas que definen duplicado.
- **`keep`**: `'first'`, `'last'` o `False` (en versiones recientes puede controlar qué filas conservar; alinear con lo visto en diagnóstico).

### 11.5 Renombrar

- **`df.rename(columns={"viejo": "nuevo"})`** o **`index={...}`**; por defecto devuelve copia salvo `inplace=True`.

### 11.6 Reemplazo vs recodificación

- **`replace`**: cambio directo `serie.replace("viejo", "nuevo")`, o **diccionario** en **todo el DataFrame** `df.replace({"Uganda": "Uga", "Ghana": "Gha"})` como en el notebook.
- **`map`**: diccionario función; las claves **no mapeadas** pasan a **NaN**. Con función, **`na_action="ignore"`** evita aplicar la función sobre NaN existentes (`df.Status.map(mayus, na_action="ignore")`).

### 11.7 Texto con `.str`

Sobre Series string-like:

- Capitalización: **`str.upper()`**, **`str.lower()`**, **`str.capitalize()`**, **`str.title()`**.
- Unir / trocear: **`str.cat()`** (concatena toda la serie; para separador entre filas ver `sep`/`na_rep` en la doc); **`str.join(sep)`** entre **caracteres** de cada string; **`str.split()`** tokeniza en listas por fila.
- **`str.len()`** longitud.
- **`str.strip()`** espacios inicio/fin (el notebook lo usa así).
- En **nombres de columnas**: **`df.columns.str.replace(pat=" ", repl="_")`**, **`str.startswith` / `str.endswith` / `str.contains`** devuelven booleanos para **`df.loc[:, mask]`**.

La cheatsheet PDF mezcla a veces “concatenar” con `split`; en pandas **`split`** separa, **`cat`** agrupa strings.

### 11.8 Patrones de los ejercicios del notebook 13

- **Memoria:** columnas `object` → `category` con `col_obj = df.select_dtypes("object").columns` y `df[col_obj] = df[col_obj].astype("category")`.
- **Nulos en fecha + moda:** contar NA → `mode().values[0]` → `fillna(moda)` → verificar `isna().sum()`.
- **Regla de negocio + borrado:** localizar índices con condición (`Loan Amount == mínimo`) y `drop(index=...)` si el volumen cumple política (en el enunciado: menos de 15 préstamos en el mínimo).
- **Nombre / mailing:** revisar NA y valores operativos (ej. **`Anonymous`** masivo); construir subset “limpio” según criterios de envío.

---

## 12. Referencia rápida — corrección (cheatsheet Pandas V)

- Categorías: `astype("category")`; ordinales: `pd.CategoricalDtype(..., ordered=True)`; reordenar: `.cat.set_categories(..., ordered=True)`.
- Numérico desde texto sucio: `pd.to_numeric(..., errors="coerce")`.
- Fechas: `pd.to_datetime(..., dayfirst=..., errors="coerce", format=...)`; mostrar: `serie.dt.strftime("...")`.
- Quitar columnas/filas: `drop(columns=...)`, `drop(index=...)`.
- Insertar columna en posición: `insert(loc, name, values)`.
- Nulos: `dropna(axis, how, subset, thresh)`; `fillna(value)`; forward/back: `ffill` / `bfill`.
- Duplicados: `drop_duplicates(subset=..., keep=...)`.
- Renombrar: `rename(columns={...}, index={...})`.
- Valores: `replace(...)`; recodificar / función: `map(..., na_action="ignore")`.
- Texto: `str.upper`, `lower`, `capitalize`, `title`, `cat`, `join`, `split`, `len`, `strip`, `replace`; sobre `columns`: `startswith`, `endswith`, `contains`.

---

## 13. Cómo seguir ampliando este documento

- Añadir **reglas de negocio** concretas (PK, invariantes, rangos válidos).
- Documentar decisiones sobre **umbrales de nulos**, **tratamiento de duplicados** y **criterios de exclusión de categorías raras**.
- Enlazar con fases siguientes (imputación, transformaciones, modelado) sin mezclar diagnóstico con “arreglo”: primero caracterizar, luego intervenir.

---

## 14. Práctica encadenada — notebooks `14_…Ejercicios` y `15_…Soluciones` (Madrid 2020)

Los notebooks **14** (plantilla con celdas para el alumno) y **15** (soluciones) cierran **Pandas V** con el mismo dataset de **accidentalidad Madrid 2020** (`00_Datasets/2020_Accidentalidad.xlsx`). La idea no es solo “limpiar”, sino **encadenar decisiones** en el mismo `df` hasta dejar el tablón listo para análisis posteriores.

### 14.1 Metodología del material

- Trabajar sobre **una copia** del notebook; el original queda como referencia.
- **Acumular** cada corrección en `df` (con `inplace=True` o reasignando columnas) para que el estado sea coherente paso a paso.
- El guion recuerda problemas ya vistos en diagnóstico: **índice de negocio** (expediente), **muchas columnas como object**, **dos últimas columnas basura**, **duplicados**, **muchos nulos**.

### 14.2 Secuencia de corrección (orden del curso)

1. **Carga:** `pd.read_excel(ruta_al_xlsx)`.
2. **Índice de negocio:** `df.set_index('Nº  EXPEDIENTE', inplace=True)` — ojo al **nombre literal** en el Excel: hay **dos espacios** entre `Nº` y `EXPEDIENTE` en la solución oficial.
3. **Quitar columnas vacías/artefacto:** `df.drop(columns=['Unnamed: 13', 'Unnamed: 14'], inplace=True)` (nombres concretos del fichero de práctica).
4. **Renombrar:** quitar el asterisco de la variable de lesividad: `df.rename(columns={'LESIVIDAD*': 'LESIVIDAD'}, inplace=True)`.
5. **Duplicados:** `df.drop_duplicates(inplace=True)` — conserva la primera aparición por defecto; la comparación es por **columnas**, no por el índice.
6. **Tipos categóricos en bloque por rango de columnas:** el enunciado pide categorizar **desde `DISTRITO` hasta `SEXO` inclusive**. Patrón útil del notebook:
   - `df.loc[:, 'DISTRITO':'SEXO']` selecciona el bloque por etiquetas.
   - Para **asignar** el resultado de `astype('category')` a **todas** esas columnas a la vez, la solución usa la lista de nombres:  
     `df[df.loc[:, 'DISTRITO':'SEXO'].columns] = df.loc[:, 'DISTRITO':'SEXO'].astype('category')`  
     Así se evita el típico choque de “slice en el LHS” sin expandir a columnas.

**Consecuencia que conviene entender:** en esa solución, **`SEXO` pasa a `category`**, pero **`ESTADO METEREOLÓGICO` queda fuera del slice** (sigue como `object` salvo que se convierta aparte). Por eso en las soluciones solo hace falta **`cat.add_categories`** explícito en **`SEXO`** al introducir `"Se desconoce"`; en **`ESTADO METEREOLÓGICO`** basta `fillna('Se desconoce')` si la columna sigue siendo **object/string**. Si en tu pipeline pasás también **meteorología** a `category`, tendrías que **añadir la categoría nueva antes del `fillna`**, igual que con `SEXO`.

7. **Nulos — siempre con criterio por variable** (el material propone una tabla de decisión):
   - **`NÚMERO` y `DISTRITO`:** muy pocos nulos → **eliminar filas** con `df.dropna(subset=['NÚMERO', 'DISTRITO'], inplace=True)`.
   - **`TIPO PERSONA`, `TIPO ACCIDENTE`, `TIPO VEHÍCULO`:** pocos nulos → **imputación por moda** por columna: `s.mode().values[0]` y luego `fillna(moda)`.
   - **`SEXO` y `ESTADO METEREOLÓGICO`:** categoría explícita **"Se desconoce"** para los NA (tras revisar niveles; en `category` → `add_categories` antes de `fillna`).
   - **`LESIVIDAD`:** los nulos se interpretan como **“sin lesión”** en el guion → `fillna(0)` (supuesto de negocio: documentarlo; no es lo mismo que “dato perdido al azar”).

8. **Verificación final:** `df.isna().sum().sort_values(ascending=False)` hasta que los conteos encajen con lo esperado; en categóricas, `value_counts(dropna=False)` para confirmar el nuevo nivel y que no queden NA.

### 14.3 Patrones reutilizables extraídos de los notebooks

- **Moda robusta en una línea por columna:** `moda = df['COL'].mode().values[0]` (si hay empate en la moda, pandas devuelve varias filas en `mode()`; `.values[0]` toma la primera — conviene saberlo en producción).
- **Inspección de niveles categóricos:** `serie.cat.categories` antes de imputar con un literal nuevo.
- **Comprobar imputaciones:** `value_counts(dropna=False)` para ver NA restantes y frecuencias del nivel añadido.

### 14.4 Lienzo con el resto del documento

Esta práctica es el **mismo hilo** que la sección 10 (diagnóstico Madrid 2020) y la sección 11 (herramientas de corrección): aquí solo se **operacionaliza** el plan en un único `df`, en un orden que respeta dependencias (por ejemplo, **tipos y duplicados antes** de estrategias de nulos que asumen categorías ya definidas).

---

*Última actualización: añadidos notebooks 14–15 (ejercicios + soluciones, pipeline Madrid 2020: índice expediente, drop columnas Unnamed, rename LESIVIDAD*, drop_duplicates, slice DISTRITO–SEXO → category, dropna subset, imputación moda, “Se desconoce”, LESIVIDAD fillna(0)). Mantiene Pandas IV, práctica diagnóstico §10, corrección §11–12 y notebook 13 Kiva.*
