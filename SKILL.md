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

### 2b. ESTADO == 0 — filtrar siempre antes de calcular
`ESTADO == 0` son casos no aplicables que distorsionan denominadores. Agregar al inicio del análisis:
```r
base <- base %>% filter(ESTADO != 0)
```

### 2c. Denominador tasa de actividad
El denominador es `ESTADO %in% c(1,2,3)` (PEA + inactivos), **NO** toda la población. Incluye menores si no se filtran antes:
```r
# Correcto
tasa_actividad <- sum(PONDERA[ESTADO %in% c(1,2)]) / sum(PONDERA[ESTADO %in% c(1,2,3)])
# Incorrecto — divide sobre toda la base
tasa_actividad <- sum(PONDERA[ESTADO %in% c(1,2)]) / sum(PONDERA)
```

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

### Clasificación TCP / TCSNP para cuenta propia (CAT_OCUP == 2)

Cuando se necesita dividir la cuenta propia en calificada vs. no calificada, usar el **5° dígito de `PP04D_COD`** (CNO 2001):

```r
cno_calific <- substr(as.character(PP04D_COD), 5, 5)

cat_cp <- case_when(
  cno_calific %in% c("1", "2") ~ "TCP",    # Profesional (1) + Técnico (2)
  cno_calific %in% c("3", "4") ~ "TCSNP",  # Operativo (3) + No calificado (4)
  TRUE                          ~ NA_character_
)
```

- **TCP** (Trabajadores Cuenta Propia calificados): calificación 1 + 2
- **TCSNP** (Cuenta Propia sin calificación/operativos): calificación 3 + 4

Esta clasificación es consistente para **ambos períodos PRE y POST 4T2023** — `PP04D_COD` no cambió con el nuevo cuestionario.

---

### Consistencia de panel — criterios de validación

Al usar `organize_panels()`, la columna `consistencia` marca individuos con seguimiento dudoso. Los criterios estándar usados en los cursos:
- `abs(CH06 - CH06_t1) > 2` → salto de edad imposible → error de registro
- `CH04 != CH04_t1` → cambio de sexo → error de registro

Estos casos **no se descartan automáticamente** — se marcan con `consistencia = FALSE` y el analista decide si incluirlos. Para análisis de transiciones laborales: filtrar solo `consistencia == TRUE`.
