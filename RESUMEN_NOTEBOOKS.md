# Preguntas Críticas para Ajustar los Prompts

## 1. ESTRUCTURA DE DATOS Y FORMATO

### 1.1 DataFrames Wide vs Long
Veo que usas formato **wide** (fechas × tickers), que es PERFECTO para vectorización. 

**Preguntas:**
- ¿Todos tus DataFrames principales mantienen este formato wide (fechas en index, tickers en columnas)?
- ¿Los DataFrames mensuales también son wide? (ej: `prices_monthly`: 433 meses × 1,289 tickers)
- ¿El `eligibility_mask` es boolean wide? (fechas_rebalanceo × tickers)

**Implicación:** Si todo es wide, la vectorización Monte Carlo será MUCHO más eficiente.

---

## 2. ELEGIBILIDAD Y UNIVERSO DINÁMICO

### 2.1 Máscara de Elegibilidad
Tienes `eligibility_mask.parquet` que indica qué tickers son elegibles cada mes.

**Preguntas:**
- ¿El eligibility_mask ya filtra por: (a) datos completos en ventana t-13 a t-1, (b) ticker en S&P500 ese mes, (c) no delisted?
- ¿O necesitas aplicar filtros adicionales en Notebook 3?
- ¿El número de activos elegibles varía por mes (651-767)?

**Implicación:** Si eligibility_mask es completo, Notebook 3 solo necesita:
```python
# Para cada fecha de rebalanceo:
elegibles = eligibility_mask.loc[fecha]  # Boolean mask
retornos_elegibles = log_returns_monthly.loc[:, elegibles]  # Solo elegibles
# Calcular z-scores solo sobre elegibles
```

---

## 3. RETORNOS MENSUALES Y VENTANAS

### 3.1 Cálculo de Momentum
Actualmente tienes `log_returns_monthly.parquet` con retornos mes a mes.

**Preguntas:**
- ¿Necesitas calcular retornos acumulados (R_12, R_6) en Notebook 3 manualmente?
- ¿O prefieres pre-calcular "rolling windows" de retornos en Notebook 2?
- ¿Los retornos mensuales ya tienen el lag de 1 mes incorporado? (ej: retorno de enero está basado en precios de diciembre a enero)

**Propuesta de optimización:**
En Notebook 2, podríamos pre-calcular:
```python
# Retornos acumulados con vectorización
returns_12m = log_returns_monthly.rolling(12).sum().shift(1)  # t-13 a t-1
returns_6m = log_returns_monthly.rolling(6).sum().shift(1)   # t-7 a t-1
```

Esto evitaría loops en Notebook 3. ¿Te parece bien?

---

## 4. MONTE CARLO Y DATOS MENSUALES

### 4.1 Matriz de Retornos para MC
Para Monte Carlo necesitamos una matriz densa de retornos mensuales.

**Preguntas clave:**
- ¿Cómo manejas los NaN en `log_returns_monthly` para Monte Carlo?
  - **Opción A:** Rellenar NaN con 0 (activo no disponible = retorno 0)
  - **Opción B:** Rellenar NaN con media del mes (conservador)
  - **Opción C:** Máscara de disponibilidad (solo seleccionar activos disponibles)

- ¿El universo de activos disponibles cada mes es el que está en `eligibility_mask`?

**Propuesta para MC vectorizado:**
```python
# En cada mes, solo considerar activos elegibles
for mes_idx in range(n_meses):
    disponibles = elegibles_mask[mes_idx]  # Boolean array
    n_disponibles = disponibles.sum()
    
    # Seleccionar 20 activos de los disponibles
    indices_disponibles = np.where(disponibles)[0]
    indices_seleccionados = np.random.choice(
        indices_disponibles, 
        size=20, 
        replace=False
    )
```

¿Te parece correcto considerar solo activos elegibles? ¿O los monos pueden seleccionar cualquier activo?

---

## 5. REBALANCEO Y PRECIOS OPEN/CLOSE

### 5.1 Datos para Ejecución
Tienes separados `prices_open_monthly` y `prices_monthly` (CLOSE).

**Preguntas:**
- ¿Los precios mensuales son del ÚLTIMO DÍA HÁBIL del mes (BME)?
- ¿Para rebalanceo en fecha T, usas:
  - OPEN del día T (último día hábil del mes T)
  - CLOSE del día T (último día hábil del mes T)
- ¿O usas OPEN del primer día del mes siguiente?

**Clarificación importante:** 
Si rebalanceas el 31/Ene, ¿vendes a OPEN del 31/Ene y compras a CLOSE del 31/Ene? ¿O vendes a OPEN del 1/Feb?

---

## 6. SESGO DE SUPERVIVENCIA - ESTRATEGIA

### 6.1 Mitigación Actual
Mencionas que tienes universo dinámico (activos entran cuando tienen 13 meses).

**Preguntas:**
- ¿Quieres mantener este approach (universo dinámico) en Notebook 2 revisado?
- ¿O prefieres filtrar a solo activos que estuvieron COMPLETOS en todo el periodo 2015-2026?
- ¿Cómo manejas activos delisted DURANTE el backtest? (ej: activo delisted en 2020)

**Propuestas:**

**Opción A - Universo Dinámico (actual):**
✓ Más realista
✓ Incluye activos nuevos
✗ Sesgo de supervivencia presente pero documentado

**Opción B - Solo Supervivientes:**
```python
# En Notebook 2, filtrar a:
supervivientes = ticker_metadata[
    (ticker_metadata['first_date'] <= '2015-01-01') &
    (ticker_metadata['last_date'] >= '2026-01-30')
]['symbol'].tolist()
```
✓ Elimina sesgo de supervivencia
✗ Menos realista (no incluye activos que entraron después)
✗ Universo más pequeño

**¿Cuál prefieres?**

---

## 7. LOOK-AHEAD BIAS - VALIDACIÓN

### 7.1 Elegibilidad en t usa solo hasta t-1
Tu validación pasó. Perfecto.

**Pregunta de confirmación:**
- ¿Cuando calculas momentum en fecha de rebalanceo `2015-01-30`, usas retornos hasta `2014-12-31` (mes anterior)?
- ¿O usas hasta `2015-01-31` (mismo mes)?

**CRÍTICO:** Debe ser hasta el mes ANTERIOR al rebalanceo.

Ejemplo:
```
Rebalanceo: 2015-01-30 (último día de enero)
Momentum 12m: suma de retornos desde 2014-02-01 hasta 2015-01-01 (12 meses, excluyendo enero 2015)
```

¿Es así tu implementación?

---

## 8. NOTEBOOK 2 - REESCRITURA

### 8.1 Qué mantener vs cambiar

**¿Qué te gustaría cambiar en Notebook 2?**
- [ ] Mejorar forward fill (¿menos agresivo?)
- [ ] Pre-calcular retornos acumulados (R_12, R_6) para Notebook 3
- [ ] Cambiar estrategia de sesgo de supervivencia
- [ ] Optimizar estructura de datos para MC
- [ ] Añadir más validaciones
- [ ] Reducir tamaño de archivos intermedios
- [ ] Otro: ___________

---

## 9. ARCHIVOS INTERMEDIOS - OPTIMIZACIÓN

### 9.1 Tamaño y Formato
Generas 13 archivos. Algunos pueden ser redundantes.

**Preguntas:**
- ¿Necesitas tanto `clean_data.parquet` (diario) como `prices_monthly.parquet`?
- ¿Notebook 3 usa datos diarios o solo mensuales?
- ¿Notebook 4 necesita datos diarios para valoración entre rebalanceos?

**Propuesta de simplificación:**
Si Notebook 3 solo usa mensuales y Notebook 4 solo usa rebalanceos mensuales, podrías eliminar algunos archivos diarios.

---

## 10. MONTE CARLO - DISEÑO ESPECÍFICO

### 10.1 Universo de Selección para Monos

**Pregunta MUY IMPORTANTE:**

Para cada mes en Monte Carlo, los monos aleatorios seleccionan de:

**Opción A:** Solo activos ELEGIBLES ese mes (según `eligibility_mask`)
- Mismo universo que la estrategia
- Comparación justa

**Opción B:** TODOS los activos disponibles (incluso si no tienen 13 meses de histórico)
- Universo más grande
- Menos fair comparison

**Opción C:** Solo activos que SOBREVIVIERON todo el periodo
- Universo fijo
- Máximo sesgo de supervivencia

**¿Cuál quieres usar?** 

Mi recomendación: **Opción A** (solo elegibles), para comparación justa.

---

## 11. ESTRUCTURA DE RETORNOS PARA MC

### 11.1 Matriz Densa

Para MC eficiente, necesitamos:
```python
# Matriz densa: (n_meses_rebalanceo, n_activos_max)
# Ejemplo: (133 meses, 1289 activos)
retornos_mc = np.array(...)  # Con NaN donde no hay datos
```

**Preguntas:**
- ¿Prefieres matriz rectangular fija (133 × 1289) con NaN?
- ¿O lista de matrices variables por mes (133 × [651...767])?

**Opción recomendada:** Matriz fija con máscara de elegibilidad, más eficiente para numpy.

---

## 12. ESTRUCTURA PROPUESTA FINAL

Basado en tu estructura, propongo:

### Notebook 2 (Revisado):
1. Carga y formato wide
2. Forward fill (tunear parámetros si necesario)
3. Elegibilidad con lag
4. **NUEVO:** Pre-cálculo de retornos acumulados (R_12, R_6)
5. **NUEVO:** Matriz densa para MC con máscara de elegibilidad
6. Validaciones de look-ahead y calidad
7. Guardar archivos optimizados

### Notebook 3 (Ajustado):
1. Cargar retornos pre-calculados
2. Loop sobre fechas de rebalanceo
3. Filtrar elegibles ese mes
4. Calcular z-scores vectorizados
5. Seleccionar top 20
6. Guardar CSV

### Notebook 5 Monte Carlo (Optimizado):
1. Cargar matriz densa de retornos
2. Cargar máscara de elegibilidad
3. Vectorización TOTAL:
```python
# Para batch completo
for batch in batches:
    # Generar selecciones respetando elegibilidad
    for mes_idx in range(n_meses):
        disponibles = eligibility_mask[mes_idx]
        # Seleccionar 20 de disponibles
    # Calcular retornos vectorizados
```

---

## RESPONDE ESTAS PREGUNTAS PRIORITARIAS:

**TOP 3 CRÍTICAS:**

1. **¿Eligibility_mask es completo o necesitas filtros adicionales en Notebook 3?**

2. **¿Monte Carlo selecciona de activos elegibles o de todos los activos disponibles?**

3. **¿Rebalanceo en 31/Ene usa OPEN del 31/Ene o del 1/Feb?**

**SECUNDARIAS:**

4. ¿Pre-calcular R_12 y R_6 en Notebook 2?

5. ¿Mantener universo dinámico o filtrar solo supervivientes?

6. ¿Qué cambiar específicamente en Notebook 2?

Una vez respondas, te genero:
- **Notebook 2 REVISADO** (optimizado para tu estructura)
- **Notebook 3 AJUSTADO** (compatible con tus DataFrames)
- **Notebook 5 MC OPTIMIZADO** (vectorización total con eligibility_mask)