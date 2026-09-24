---
name: eph
description: >
  Diseño de registro y metodología de la Encuesta Permanente de Hogares (EPH) del INDEC.
  Lee automáticamente el archivo de diseño correcto y aplica orientación metodológica
  antes de generar scripts R o Python con microdatos EPH.
  Trigger: cuando el usuario pide un script R o Python para procesar microdatos de la EPH,
  menciona variables EPH (CODUSU, PONDERA, ESTADO, etc.), o pide análisis de bases
  usu_hogar / usu_individual.
license: Apache-2.0
metadata:
  author: claudiodavi
  version: "1.0"
---

## Cuando usar este skill

Cargar cuando el usuario:
- Pide un script en R o Python para procesar microdatos de la EPH
- Menciona variables como CODUSU, PONDERA, ESTADO, PP03C, INGOT, CH04, PP04D_COD, etc.
- Hace referencia a las bases `usu_hogar` o `usu_individual`
- Pregunta sobre la estructura de las bases EPH del INDEC
- Pide calcular tasas de empleo, desempleo, subocupación, ingresos o informalidad
- Usa los paquetes `eph` (R) o `pyeph` (Python)
- Pregunta sobre ocupaciones, clasificación de ocupaciones, CNO, o la variable PP04D_COD

---

## REGLA CRÍTICA — Nunca escribir código sin leer el diseño de registro

**Antes de escribir cualquier línea de código que referencie variables de la EPH:**
1. Determinar el archivo de diseño correcto (Step 1)
2. Leerlo con el tool Read (Step 2)
3. Solo entonces generar el código (Step 3)

Los nombres de variables, valores codificados y longitudes son exactos en esos archivos.
Cualquier código escrito de memoria puede ser incorrecto.

---

## Step 1 — Determinar qué archivo de diseño leer

### ¿Qué tipo de encuesta?

| La consulta menciona...              | Tipo             |
|:-------------------------------------|:-----------------|
| "Total Urbano" / "tot urbano"        | EPH Total Urbano |
| Sin mención especial (default)       | EPH Continua     |

### ¿Qué período?

| Período de los datos                          | Cuándo aplica        |
|:----------------------------------------------|:---------------------|
| **PRE 4T2023** | Cualquier trimestre ANTES del 4T2023 (ej: 1T2016, 3T2023) |
| **POST 4T2023** | 4T2023 en adelante (4T2023, 1T2024, 2T2024...)            |

⚠️ Si el usuario no especificó el período, **preguntar antes de continuar**.

### Tabla de decisión → archivo a leer

| Tipo           | Período     | Archivo                                                      |
|:---------------|:------------|:-------------------------------------------------------------|
| EPH Continua   | PRE 4T2023  | `~/.claude/skills/eph/assets/design/EPH_PRE_4T2023.md`      |
| EPH Continua   | POST 4T2023  | `~/.claude/skills/eph/assets/design/EPH_POST_4T2023.md`      |
| Total Urbano   | PRE 4T2023  | `~/.claude/skills/eph/assets/design/EPH_TotUrbano_PRE_4T2023.md` |
| Total Urbano   | POST 4T2023  | `~/.claude/skills/eph/assets/design/EPH_TotUrbano_POST_4T2023.md` |

---

## Step 2 — Leer el archivo con el tool Read

Usar el tool **Read** con el path absoluto determinado arriba. Leer el archivo completo.

---

## Step 3 — Aplicar reglas críticas (siempre activas)

Antes de generar código, verificar estas 4 condiciones:

### 1. Quiebre de serie 4T2023
Si los datos cruzan el 4T2023 (ej: serie 2019-2024), advertir que la serie no es
estrictamente comparable sin empalme. Ver detalles:
`~/.claude/skills/eph/assets/methodology/quiebre_serie_4t2023.md`

### 2. Ponderadores
¿El código usa ponderadores? Si no, recordar que la EPH es muestral y requiere ponderar.
- Personas → `PONDERA`
- Hogares → `PONDIH`
- Ingresos de ocupación principal → `PONDIIO`

### 2b. ESTADO == 0 — entrevista individual no realizada

`ESTADO == 0` son casos sin cuestionario individual. Para análisis de **subgrupos** (por rama,
sexo, nivel educativo, etc.) conviene excluirlos, porque no tienen las variables que definen el
corte y ensucian el denominador de ese subgrupo:

```r
base <- base %>% filter(ESTADO != 0)
```

⚠️ **NO aplicar este filtro al denominador de la tasa de actividad ni de empleo** — ver 2c: ahí el
denominador es la población total, la base completa. Filtrar antes de calcular TA/TE las infla.

*(No verificado: si el total de 30,0 M que publica el INDEC incluye o no a los `ESTADO == 0`. Son
pocos casos, así que el efecto es chico, pero si se busca reproducir la cifra publicada al decimal
conviene probar las dos variantes contra el cuadro 1.1.)*

### 2c. Denominador de tasa de actividad y de empleo — es TODA la población

El INDEC define la TA como *"la población económicamente activa (PEA) sobre el **total de la
población**"* y la TE como *"la proporción de personas ocupadas con relación a la **población
total**"*. El denominador es la base entera, **sin filtrar por `ESTADO`** — incluye a los menores
de 10 años (`ESTADO == 4`).

```r
# Correcto — denominador = toda la base
tasa_actividad <- sum(PONDERA[ESTADO %in% c(1,2)]) / sum(PONDERA) * 100
tasa_empleo    <- sum(PONDERA[ESTADO == 1])        / sum(PONDERA) * 100

# Incorrecto — excluir menores/no respuesta infla la tasa varios puntos
tasa_actividad <- sum(PONDERA[ESTADO %in% c(1,2)]) / sum(PONDERA[ESTADO %in% c(1,2,3)])
```

**Comprobación (3T2025, 31 aglomerados):** PEA 14,6 M ÷ población total 30,0 M = **48,6%**, que es
exactamente la tasa publicada. Con el denominador restringido a `ESTADO %in% c(1,2,3)` da ~57%.

La tasa de **desocupación** sí lleva denominador restringido: es sobre la PEA, no sobre la población.

```r
tasa_desocupacion <- sum(PONDERA[ESTADO == 2]) / sum(PONDERA[ESTADO %in% c(1,2)]) * 100
```

**Valores de referencia para validar (31 aglomerados urbanos):**

| Tasa | 3T2024 | 4T2024 | 1T2025 | 2T2025 | 3T2025 |
|------|--------|--------|--------|--------|--------|
| Actividad | 48,3 | 48,8 | 48,2 | 48,1 | 48,6 |
| Empleo | 45,0 | 45,7 | 44,4 | 44,5 | 45,4 |
| Desocupación abierta | 6,9 | 6,4 | 7,9 | 7,6 | 6,6 |
| Subocupación | 11,4 | 11,3 | 10,0 | 11,6 | 10,9 |

Fuente: INDEC, *Mercado de trabajo. Tasas e indicadores socioeconómicos (EPH)*, 3T2025, cuadro 1.1.

### 2c-bis. Informalidad — el denominador excluye los ignorados

`EMPLEO == 9` es la categoría **"Ocupados con ignorado en la formalidad"** (Metodología INDEC
N° 43, cuadro 4), no un dato faltante. `!is.na(EMPLEO)` **no lo filtra**. El INDEC aclara al pie
de sus cuadros: *"no se incluye personas sin información respecto a la condición de formalidad
en el empleo"*.

```r
# Correcto — excluye explícitamente el ignorado
base |> filter(ESTADO == 1, EMPLEO %in% c(1, 2))
```

Ídem `SECTOR`: usar `SECTOR %in% c(1, 2, 3)`.

**Referencia (tasa de empleo informal):** 4T2023 41,4% · 4T2024 42,0% · 4T2025 43,0%.
Por rama, 4T2025: servicio doméstico 78,0% · construcción 73,8% · comercio 52,6% ·
industria manufacturera 37,2%.

### 2d. Tasa vs conteo — nunca mezclar

**TASA** = cociente entre 0 y 1 (o porcentaje entre 0 y 100). Tiene denominador.
**CONTEO** = suma ponderada, resultado en personas (millones o miles).

```r
# TASA de desocupación — resultado: ~7.7 (porcentaje)
tasa_desoc <- sum(PONDERA[ESTADO==2]) / sum(PONDERA[ESTADO %in% c(1,2)]) * 100

# CONTEO de desocupados — resultado: ~1,088,000 (personas)
n_desoc <- sum(PONDERA[ESTADO==2])
```

❌ **Error grave**: "La tasa de desocupación fue de 994,334 personas" — mezcla tasa con conteo.
Si la pregunta dice "tasa", "porcentaje" o "%" → resultado debe ser un número entre 0 y 100.
Si dice "cuántas personas" o "cantidad" → resultado en personas (suma ponderada).

### 2e. Serie de N trimestres — iterar correctamente

Cuando la pregunta pide "los últimos N trimestres" o una serie temporal:

```r
# Correcto — lista explícita de períodos
periodos <- list(
  list(year=2023, period=3),
  list(year=2023, period=4),
  list(year=2024, period=1),
  list(year=2024, period=2)
)

bases <- lapply(periodos, function(p) {
  b <- get_microdata(year=p$year, period=p$period, type="individual")
  b$ANO4 <- p$year
  b$TRIMESTRE <- p$period
  b
})
base_total <- bind_rows(bases)
```

❌ **Error común**: vectores `anos` y `trimestres` de distinta longitud iterar uno sobre otro. Siempre usar lista de pares (year, period).

### 2f. 🔴 Desborde de enteros al ponderar ingresos — OBLIGATORIO `as.numeric()`

Las variables de ingreso (`P21`, `P47T`, `ITF`, `IPCF`, `V*_M`, `PP06C`…) y los ponderadores llegan
de la base como **`integer`**. Su producto desborda int32 (>2.147.483.647): con ponderadores de
1.000–2.500, eso ocurre en **ingresos desde ~$860.000**. R convierte el desborde en `NA`, y
`na.rm = TRUE` descarta esas filas **en silencio**.

**No es un aviso cosmético: cambia el número.** El sesgo elimina justo a los que más ganan.

```r
# Correcto
sum(as.numeric(P21) * PONDIIO, na.rm = TRUE) / sum(PONDIIO, na.rm = TRUE)

# Incorrecto — desborda, sesga hacia abajo y el resultado parece plausible
sum(P21 * PONDIIO, na.rm = TRUE) / sum(PONDIIO, na.rm = TRUE)
```

**Caso real (4T2025):** sin castear, la brecha salarial de género dio 6,6% con un ingreso medio de
~$440.000. Con `as.numeric()`, dio **29,6%** con $838.336 (mujeres) y $1.191.364 (varones) — que son
**exactamente** las cifras publicadas por el INDEC.

### 2g. Nombrar SIEMPRE el indicador usado

Muchas preguntas admiten más de un indicador defendible, y el número cambia mucho según cuál se
elija. Un número sin etiqueta es ambiguo aunque el cálculo esté bien.

- Nombrá el indicador en palabras **y** con la variable: *"el ingreso de la ocupación principal (`P21`)"*.
- Si hay una segunda lectura defendible sobre la misma base, calculá y reportá las dos.
- No inventes una disyuntiva donde hay un solo indicador.

| La pregunta suena a… | Indicador A | Indicador B |
|---|---|---|
| Ingresos de una persona | Ocupación principal (`P21`, `PONDIIO`) | Total individual (`P47T`, `PONDII`) |
| Ingresos de un hogar | Ingreso total familiar (`ITF`) | Per cápita familiar (`IPCF`) |
| Informalidad | Del **empleo** (`EMPLEO`) | De la **unidad económica** (`SECTOR`) |
| Falta de trabajo | Tasa de desocupación | Presión sobre el mercado (subocupados + ocupados demandantes) |
| Cobertura geográfica | EPH 31 aglomerados (trimestral) | EPH Total Urbano (anual) |

⚠️ La **brecha de género que publica el INDEC** se calcula sobre el **ingreso individual**
(`P47T`, ponderador `PONDII`), no sobre la ocupación principal.

### 2g-bis. CRITERIO GENERAL — universo, ponderador y casos descartados

Aplica a **TODO** cálculo sobre la EPH, no solo a ingresos. Tres decisiones definen cualquier
indicador, y equivocarse en una cambia el número **sin dar ningún error**.

**1. El universo.** No es "todas las filas de la base": cada indicador tiene el suyo.

| Familia | Indicador | Universo | Filtro en R | Ponderador |
|---|---|---|---|---|
| **Mercado de trabajo** | Tasa de actividad | Toda la población | (sin filtro) | `PONDERA` |
| | Tasa de empleo | Toda la población | (sin filtro) | `PONDERA` |
| | Tasa de desocupación | PEA | `ESTADO %in% c(1,2)` | `PONDERA` |
| | Tasa de subocupación | PEA | `ESTADO %in% c(1,2)` | `PONDERA` |
| **Informalidad** | Del empleo | Ocupados con condición conocida | `ESTADO == 1, EMPLEO %in% c(1,2)` | `PONDERA` |
| | De la unidad económica | Ocupados con sector conocido | `ESTADO == 1, SECTOR %in% c(1,2,3)` | `PONDERA` |
| **Ingresos** | Ocupación principal (`P21`) | Ocupados con ingreso | `ESTADO == 1, P21 > 0` | `PONDIIO` |
| | Total individual (`P47T`) | Todos los perceptores | `P47T > 0` | `PONDII` |
| | Per cápita familiar (`IPCF`) | Toda la población | (sin filtro) | `PONDIH` |
| | Ingreso total familiar (`ITF`) | Hogares | nivel hogar | `PONDIH` |

**Regla de olfato**: un indicador *del puesto de trabajo* lleva filtro por condición de actividad;
uno *de la persona* o *del hogar*, no. `P47T` es de la persona — restringirlo a `ESTADO == 1`
excluye jubilados, rentistas y perceptores de transferencias, sube el promedio y deja de coincidir
con lo publicado.

**2. El ponderador** corresponde a la familia de la variable, no a la unidad de análisis que uno
imagina. Ver 2 arriba.

**3. Ningún caso se descarta por accidente.** Todo caso excluido tiene que ser una decisión
explícita, nunca un efecto colateral. Las tres formas en que la EPH pierde casos en silencio:

- **Ignorados codificados**: `EMPLEO == 9`, `SECTOR == 9` son *categorías*, no `NA` — `!is.na()` no
  los filtra. Excluirlos por lista de valores válidos.
- **Desborde de enteros** (ver 2f): `ingreso * ponderador` sin `as.numeric()` da `NA`, y
  `na.rm = TRUE` lo descarta.
- **Filtros de más**: restringir el universo por costumbre donde el indicador no lo pide.

**4. Contrastar contra lo publicado.** Si existe cifra oficial del INDEC para ese indicador y
período, el resultado debe coincidir. Si difiere, el error está en el cálculo — revisar universo,
ponderador y casos descartados, en ese orden.

Comprobación 4T2025: con el universo correcto, `P47T` da $838.336 (mujeres) y $1.191.364 (varones),
exactamente las cifras del INDEC. Restringido a ocupados da ~$1.037.877 y ~$1.325.689, que no
corresponden a ningún indicador publicado.

### 2h. Valores de referencia — ingresos 4T2025 (31 aglomerados)

| Indicador | Valor | Ponderador |
|-----------|-------|------------|
| Ingreso medio de la ocupación principal | **$1.068.540** | `PONDIIO` |
| Mediana de la ocupación principal | $800.000 | `PONDIIO` |
| Ingreso medio de asalariados | $1.082.635 | `PONDIIO` |
| — con descuento jubilatorio | $1.321.353 | `PONDIIO` |
| — sin descuento jubilatorio | $651.484 | `PONDIIO` |
| Ingreso individual medio (perceptores) | **$1.011.863** | `PONDII` |
| — varones | **$1.191.364** | `PONDII` |
| — mujeres | **$838.336** | `PONDII` |
| Ingreso per cápita familiar medio | $635.996 | `PONDIH` |
| Mediana del IPCF | $450.000 | `PONDIH` |
| Coeficiente de Gini del IPCF | 0,427 | `PONDIH` |

Fuente: INDEC, *Evolución de la distribución del ingreso (EPH)*, 4T2025.

> Si un cálculo de ingresos da muy por debajo de estos valores, sospechar del desborde de enteros
> (2f) antes que de los datos.

### 3. Merge Hogar/Personas
Si el código une las dos bases, verificar que use las 4 keys:
`CODUSU`, `NRO_HOGAR`, `ANO4`, `TRIMESTRE`

### 4. EPH Continua vs Total Urbano
Si el análisis compara o combina ambas fuentes, advertir que no son comparables directamente
(distinta cobertura geográfica). Ver:
`~/.claude/skills/eph/assets/methodology/eph_continua_vs_total_urbano.md`

---

## Step 4 — Cargar metodología adicional bajo demanda

Leer el archivo correspondiente solo si la consulta lo requiere:

| La consulta involucra...                         | Leer este archivo                    |
|:-------------------------------------------------|:-------------------------------------|
| Tasas de actividad, empleo, desocupación, subocupación | `assets/methodology/indicadores_mercado_laboral.md` |
| Confiabilidad por aglomerado, CV, errores de muestreo   | usar `eph::errores_muestrales` (tabla built-in del paquete) |
| Cómo ponderar, qué ponderador usar               | `assets/methodology/ponderadores.md` |
| Series de tiempo pre/pos 4T2023                  | `assets/methodology/quiebre_serie_4t2023.md` |
| Diferencias entre EPH continua y Total Urbano    | `assets/methodology/eph_continua_vs_total_urbano.md` |
| Panel rotante, seguimiento de individuos         | `assets/methodology/panel_rotante.md` |
| Informalidad laboral, empleo no registrado       | `assets/methodology/informalidad_laboral.md` |

Los paths completos son `~/.claude/skills/eph/assets/methodology/<archivo>`.

---

## Step 5 — Cargar referencia de packages y clasificadores bajo demanda

Leer solo cuando la consulta específicamente lo requiera:

| La consulta involucra...                                          | Leer este archivo                                      |
|:------------------------------------------------------------------|:-------------------------------------------------------|
| Funciones del paquete `eph` de R (`get_microdata`, `organize_cno`, etc.) | `assets/tools/eph_package_r.md`             |
| Funciones de `pyeph` de Python (`pyeph.get`, `LaborMarket`, etc.)        | `assets/tools/pyeph_package.md`             |
| Clasificación de ocupaciones, variable PP04D_COD, CNO 2001                | `assets/classifiers/cno_2001.md`            |

Los paths completos son `~/.claude/skills/eph/assets/tools/<archivo>` o `~/.claude/skills/eph/assets/classifiers/<archivo>`.

**Cuándo leer eph_package_r.md:**
- El usuario pide un script R con `eph`, `get_microdata()`, `organize_panels()`, `calculate_poverty()`, `organize_cno()`
- El usuario no sabe qué funciones tiene el paquete

**Cuándo leer pyeph_package.md:**
- El usuario pide un script Python con `pyeph`, `pyeph.get()`, `LaborMarket`, `Poverty`
- El usuario no sabe la API de pyeph

**Cuándo leer cno_2001.md:**
- El usuario pide clasificar o interpretar ocupaciones
- El usuario menciona PP04D_COD, CNO, `organize_cno()`, o calificación/jerarquía ocupacional
- El usuario quiere saber qué valor de PP04D_COD corresponde a qué ocupación

---

## Packages recomendados

| Lenguaje | Package          | Repositorio                                  | Doc local |
|:---------|:-----------------|:---------------------------------------------|:----------|
| R        | `eph` (ropensci) | https://github.com/ropensci/eph              | `assets/tools/eph_package_r.md` |
| Python   | `pyeph`          | https://github.com/institutohumai/pyeph      | `assets/tools/pyeph_package.md` |

---

## Advertencias que incluir siempre en el output

Agregar al final de cada script generado las advertencias que apliquen:

- **Quiebre 4T2023:** si la serie cruza ese período
- **Ponderadores:** si el código no pondera explícitamente
- **Cobertura:** en subtítulos y referencias usar "EPH 31 aglomerados urbanos" (EPH trimestral) o "EPH Total Aglomerados Urbanos" (Total Urbano). No decir "EPH continua, 31 aglomerados urbanos" — la palabra "continua" se omite.
- **CODUSU no es único por trimestre:** aclarar si el usuario arma un panel

---

## Notas metodológicas confirmadas en uso real

### Caching con destfile en get_microdata()

Siempre sugerir `destfile` para evitar re-descargar (100s → 0.5s):
```r
base <- get_microdata(
  year = 2023, period = 1,
  type = "individual",
  vars = c("PONDERA", "ESTADO", "CAT_OCUP", "AGLOMERADO"),
  destfile = "data/base_cache.rds"
)
```
Si el archivo ya existe, `get_microdata()` lo lee desde disco.

---

### Errores de muestreo — aglomerados chicos poco confiables

La tabla `eph::errores_muestrales` tiene CV por aglomerado y período. Aglomerados con alta CV (>20%) son poco confiables a nivel trimestral. Estrategias:
- Poolear 2 trimestres consecutivos
- Usar base anual / semestral
- Reportar junto al estimado el CV o el intervalo de confianza

```r
library(eph)
errores_muestrales  # tabla built-in
```

---

### Jubilados activos — no filtrar por CAT_INAC

**NUNCA** usar `CAT_INAC == 1` como única condición para identificar perceptores de jubilación/pensión.
`CAT_INAC == 1` captura solo a los **inactivos** que se declaran jubilados/pensionados,
pero excluye a quienes **cobran una jubilación o pensión y además trabajan** (ocupados o activos en el mercado laboral).

El filtro correcto para identificar a todos los perceptores de ingreso jubilatorio es:

```r
# PRE 4T2023
filter(V2_M > 0)

# POST 4T2023
filter(V2_01_M + V2_02_M + V2_03_M > 0)

# O bien, después de construir la variable armonizada ing_jub:
filter(ing_jub > 0)
```

Usar `CAT_INAC == 1` solo si el análisis busca **específicamente** describir la situación
de los inactivos que se autoidentifican como jubilados (ej: composición del grupo inactivo),
no para calcular el ingreso jubilatorio promedio o la cobertura previsional.

---

### Clasificación TCP / TCP NP para cuenta propia (CAT_OCUP == 2)

Para dividir la cuenta propia en profesional vs. no profesional se usa la **calificación ocupacional**: el **5° dígito de `PP04D_COD`** (CNO 2001), o `clase1` en las bases procesadas del CIAS (coinciden 1 a 1).

```r
# PP04D_COD puede venir numérico (.sav, read.table): 5002 es "05002".
# Rellenar a 5 dígitos ANTES de cortar, o substr() toma el dígito equivocado.
cno_calific <- substr(sprintf("%05d", as.integer(PP04D_COD)), 5, 5)

cat_cp <- case_when(
  cno_calific == "1"                ~ "TCP",     # Profesional
  cno_calific %in% c("2", "3", "4") ~ "TCP NP",  # Técnico + Operativo + No calificado
  TRUE                              ~ NA_character_
)
```

- **TCP** (cuenta propia profesional): **solo** calificación 1.
- **TCP NP** (cuenta propia no profesional): calificación 2 + 3 + 4 — los **técnicos van acá**, no con los profesionales.

⚠️ Una versión anterior de esta nota agrupaba profesional + técnico como "TCP". Estaba mal: infla el
grupo profesional (3,3% vs 1,3% de la población) y le triplica la pobreza (~19-24% vs ~7%, 3T2025-1T2026).

Esta clasificación es consistente para **ambos períodos PRE y POST 4T2023** — `PP04D_COD` no cambió con el nuevo cuestionario.

---

### Consistencia de panel — criterios de validación

Al usar `organize_panels()`, la columna `consistencia` marca individuos con seguimiento dudoso. Los criterios estándar usados en los cursos:
- `abs(CH06 - CH06_t1) > 2` → salto de edad imposible → error de registro
- `CH04 != CH04_t1` → cambio de sexo → error de registro

Estos casos **no se descartan automáticamente** — se marcan con `consistencia = FALSE` y el analista decide si incluirlos. Para análisis de transiciones laborales: filtrar solo `consistencia == TRUE`.
