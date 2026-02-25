# Resumen Completo del Proyecto de Backtesting - Estrategia Momentum

## Índice

1. [Notebook 1: Carga de Datos](#notebook-1-carga-de-datos)
2. [Notebook 2: EDA y Preparación de Datos](#notebook-2-eda-y-preparación-de-datos)
3. [Notebook 3: Implementación de Estrategia Momentum](#notebook-3-implementación-de-estrategia-momentum)
4. [Notebook 4: Motor de Backtesting](#notebook-4-motor-de-backtesting)
5. [Notebook 5: Comparativa y Métricas](#notebook-5-comparativa-y-métricas)
6. [Decisiones Arquitectónicas Clave](#decisiones-arquitectónicas-clave)

---

## Notebook 1: Carga de Datos

### Objetivo Principal

Este notebook se encarga exclusivamente de descargar y almacenar los datos necesarios para toda la práctica. Su propósito es preparar el dataset histórico del S&P 500 y el benchmark SPY en formatos estructurados y listos para procesamiento.

### Flujo de Trabajo Paso a Paso

1. **Imports y Configuración**
   - Importa librerías: `pandas`, `pyarrow`, `yfinance`
   - Configura constantes: directorios de datos, ticker del benchmark (SPY), fecha de inicio (2015-01-01)
   - Define nombre del archivo parquet a buscar

2. **Verificación del Archivo desde Google Drive**
   - Implementa función `find_parquet()` que busca el archivo en múltiples ubicaciones posibles
   - Valida que el archivo existe antes de continuar
   - El archivo debe descargarse manualmente desde Google Drive y colocarse en `datos/raw/` con el nombre `sp500_history.parquet`

3. **Carga del Dataset**
   - Función `load_parquet_dataset()` carga el dataset usando pyarrow directamente
   - Convierte columna 'date' a datetime y la usa como índice
   - Ordena por fecha
   - Valida estructura básica: shape, columnas, rango de fechas, número de tickers únicos
   - Muestra información detallada del dataset (tipos de datos, primeras/últimas filas)

4. **Descarga del Benchmark**
   - Función `download_benchmark_data()` descarga datos de SPY desde yfinance
   - Formatea datos con la misma estructura que el dataset principal
   - Aplica mismo formato de columnas para consistencia
   - Guarda benchmark en `spy_data.parquet`

5. **Guardado del Dataset Completo**
   - Guarda dataset completo procesado en `tickers_data.parquet`
   - Usa pyarrow directamente para eficiencia
   - Genera resumen final con información del dataset

### Decisiones Técnicas Clave

- **Formato Parquet**: Se usa formato Parquet en lugar de CSV por eficiencia de almacenamiento y preservación de tipos de datos
- **Estructura de datos**: El dataset mantiene estructura long format con columnas: `symbol`, `assetid`, `security_name`, `sector`, `industry`, `subsector`, `in_sp500`, `open`, `high`, `low`, `close`, `volume`, `unadjusted_close`
- **Índice temporal**: Se usa la columna 'date' como índice DatetimeIndex para facilitar operaciones temporales
- **Benchmark SPY**: Se descarga desde yfinance para garantizar datos actualizados y consistentes con el mercado
- **Validación robusta**: Búsqueda en múltiples rutas posibles para el archivo parquet, facilitando ejecución desde diferentes directorios

### Inputs y Outputs

**Inputs:**
- Archivo `sp500_history.parquet` descargado manualmente desde Google Drive

**Outputs:**
- `sp500_history.parquet`: Archivo original sin procesar (en `datos/raw/`)
- `spy_data.parquet`: Datos del benchmark SPY (en `datos/raw/`)
- `tickers_data.parquet`: Dataset completo procesado y listo para limpieza (en `datos/raw/`)

**Características del Dataset:**
- Total de tickers: 1,289
- Rango de fechas: 1990-01-02 a 2026-01-30
- Total de registros: 7,250,110
- 14 columnas por registro

### Dependencias

- **Librerías externas**: `pandas`, `pyarrow`, `yfinance`
- **Archivos externos**: Dataset histórico del S&P 500 desde Google Drive
- **Sin dependencias de otros notebooks**: Este es el punto de entrada del pipeline

---

## Notebook 2: EDA y Preparación de Datos

### Objetivo Principal

Transforma datos diarios en estructuras mensuales optimizadas para la implementación de estrategia (Notebook 3), el motor de backtesting (Notebook 4) y el análisis de Monte Carlo (Notebook 5). **CRÍTICO**: Todos los DataFrames están en formato WIDE (fechas × tickers) para operaciones vectorizadas eficientes.

### Flujo de Trabajo Paso a Paso

#### PARTE 1: Configuración e Imports
- Define constantes críticas:
  - `HISTORY_START = '2013-12-01'`: 13 meses antes de BACKTEST_START para calcular momentum en primera fecha
  - `BACKTEST_START = '2015-01-01'`: Inicio del backtesting
  - `FFILL_LIMIT = 5`: Límite conservador para forward fill
  - `LAG_MONTHS = 1`: Lag para evitar reversión a la media

#### PARTE 2: Carga y Estructuración Inicial
- Carga `tickers_data.parquet` y `spy_data.parquet`
- Filtra datos desde `HISTORY_START` (optimiza memoria, reduce de ~7M a ~2M filas)
- Elimina timezone del índice para consistencia
- **Construye DataFrames WIDE**: Convierte formato long a wide usando `pivot_table()`
  - `prices_close_wide`: Precios CLOSE (fechas × tickers)
  - `prices_open_wide`: Precios OPEN (fechas × tickers)
  - `in_sp500_wide`: Flag de pertenencia a S&P 500 (fechas × tickers)

#### PARTE 3: Análisis de Disponibilidad Temporal
- Calcula `first_date` y `last_date` por ticker
- Identifica delistings (activos que terminaron antes del final del dataset)
- Calcula completitud dentro de la vida del activo (no sobre todo el dataset)
- Construye `ticker_metadata` con información completa
- Visualiza distribución de delistings y fechas de inicio

#### PARTE 4: Forward Fill Controlado
- Aplica forward fill con límite de 5 días consecutivos
- **Solo dentro de vida del activo**: No extiende vida más allá de `last_date` (evita look-ahead)
- Aplica máscara post-ffill para eliminar precios después de `last_date`
- Documenta razones para NaNs mantenidos (suspensiones, errores estructurales)

#### PARTE 5: Agregación Mensual
- Agrega precios diarios a frecuencia mensual usando **Business Month End (BME)**
- Usa método 'last' para tomar último precio disponible del mes
- Valida que todas las fechas son BME (último día hábil del mes)
- Genera: `prices_monthly_close`, `prices_monthly_open`, `in_sp500_monthly`

#### PARTE 6: Cálculo de Retornos Logarítmicos
- Calcula retornos logarítmicos mensuales: `r_log = ln(P_t / P_t-1)`
- Valida que no hay infinitos
- Valida rango razonable de retornos
- Visualiza distribución de retornos
- Documenta outliers extremos (ej: SBNY durante crisis bancaria 2023)

#### PARTE 7: Calendario de Rebalanceo
- Filtra fechas mensuales desde `BACKTEST_START`
- Genera `rebalance_dates`: 133 fechas de rebalanceo mensuales
- Valida consistencia (133 meses esperados vs reales)

#### PARTE 8: Pre-cálculo de Retornos Acumulados
- **Optimización crítica**: Pre-calcula R_12 y R_6 para todas las fechas
- `returns_12m`: Ventana móvil de 12 meses con `shift(1)` para lag
- `returns_6m`: Ventana móvil de 6 meses con `shift(1)` para lag
- Alinea con `rebalance_dates`: `returns_12m_rebal`, `returns_6m_rebal`
- Valida look-ahead: R_12 usa retornos desde t-13 a t-1, NO usa mes actual

#### PARTE 9: Construcción de Eligibility Mask
- Función `build_eligibility_mask()` construye máscara booleana
- **Criterios de elegibilidad en fecha t**:
  1. Precios disponibles en t-1, t-7, t-13 (necesarios para R_12 y R_6)
  2. Precios > 0
  3. Activo en S&P 500 en t-1 (usa lag, evita look-ahead)
  4. Activo NO delisted antes de t
- **CRÍTICO**: Solo usa datos hasta t-1, nunca t o futuro
- Resultado: `eligibility_mask` (133 fechas × 845 tickers, boolean)

#### PARTE 10: Validación de Look-Ahead Bias
- Test automatizado `validate_no_lookahead()` verifica que elegibilidad no usa información futura
- Valida en primeras 10 fechas de rebalanceo
- Si detecta violaciones, detiene pipeline con error explícito
- **Resultado**: Test PASADO (ninguna violación detectada)

#### PARTE 11: Procesamiento de SPY (Benchmark)
- Aplica mismo pipeline que a tickers individuales:
  - Forward fill (límite 5 días)
  - Agregación a BME mensual
  - Retornos logarítmicos mensuales
- Alinea con `rebalance_dates`
- Genera: `spy_prices_monthly`, `spy_log_returns_monthly`

#### PARTE 12: Guardado de Archivos Esenciales
- Guarda 11 archivos en formato optimizado:
  - **Para Notebook 3**: `returns_12m_rebal.parquet`, `returns_6m_rebal.parquet`, `eligibility_mask.parquet`, `rebalance_dates.csv`, `prices_monthly_close.parquet`
  - **Para Notebook 4**: `prices_monthly_open.parquet`
  - **Para Notebook 5**: `spy_prices_monthly.parquet`, `spy_log_returns_monthly.parquet`, `ticker_metadata.csv`, `log_returns_monthly.parquet`

#### PARTE 13: Resumen y Validaciones Finales
- Resumen ejecutivo con estadísticas clave
- Documenta sesgo de supervivencia (inherente al S&P 500, no introducido por estrategia)
- Valida look-ahead bias: PASADO
- Valida formato WIDE: VERIFICADO
- Valida alineación temporal: VERIFICADO

### Decisiones Técnicas Clave

#### Formato WIDE (fechas × tickers)
- **Justificación**: Permite operaciones vectorizadas eficientes con pandas/numpy
- **Ventajas**: Acceso directo por fecha usando `.loc[fecha]`, cálculo vectorizado de retornos acumulados, filtrado eficiente
- **Alternativa rechazada**: Formato LONG requeriría loops costosos

#### Forward Fill Controlado (FFILL_LIMIT = 5)
- **Justificación**: Rellena gaps pequeños (holidays, suspensiones temporales) sin extender artificialmente vida de activos delisted
- **Límite conservador**: 5 días balancea rellenar gaps legítimos vs evitar sesgo de supervivencia
- **Máscara de vida**: Post-ffill elimina precios después de `last_date`, previniendo look-ahead

#### Agregación a Business Month End (BME)
- **Justificación**: Garantiza que precios sean observables y ejecutables (último día hábil del mes)
- **Método 'last'**: Toma último precio disponible del mes, apropiado para rebalanceos de fin de mes
- **Validación**: Verifica que todas las fechas son BME

#### Retornos Logarítmicos
- **Justificación**: Son aditivos en el tiempo (r_total = r1 + r2 + ...), permitiendo calcular momentum acumulado sumando retornos mensuales
- **Ventajas**: Más eficientes computacionalmente que retornos aritméticos, simétricos, mejor aproximan distribuciones normales

#### Lag de 1 Mes (LAG_MONTHS = 1)
- **Justificación**: Evita ruido de reversión a la media (short-term reversal) que afecta típicamente al mes inmediatamente posterior
- **Implementación**: `shift(1)` en ventanas móviles excluye mes actual del cálculo de momentum
- **Validación**: R_12 usa retornos desde t-13 a t-1, NO usa mes actual

#### Pre-cálculo de Retornos Acumulados
- **Justificación**: Optimización crítica que elimina loops costosos en Notebook 3
- **Ventajas**: Reduce tiempo de ejecución en órdenes de magnitud, permite análisis iterativos rápidos
- **Implementación**: Ventanas móviles vectorizadas con pandas

#### Eligibility Mask con Criterios Estrictos
- **Requisitos**: Precios en t-1, t-7, t-13 + flag S&P 500 en t-1 + activo no delisted
- **Justificación**: Garantiza que todos los activos elegibles tengan suficiente histórico para calcular momentum
- **Sin look-ahead**: Solo usa datos hasta t-1, validado con test automatizado

### Inputs y Outputs

**Inputs:**
- `tickers_data.parquet`: Dataset completo de tickers (desde Notebook 1)
- `spy_data.parquet`: Datos del benchmark SPY (desde Notebook 1)

**Outputs (11 archivos):**

**Para Notebook 3:**
- `prices_monthly_close.parquet`: Precios CLOSE fin de mes (146 meses × 845 tickers)
- `returns_12m_rebal.parquet`: Retornos acumulados 12M pre-calculados (133 fechas × 845 tickers)
- `returns_6m_rebal.parquet`: Retornos acumulados 6M pre-calculados (133 fechas × 845 tickers)
- `eligibility_mask.parquet`: Máscara boolean de elegibilidad (133 fechas × 845 tickers)
- `rebalance_dates.csv`: Calendario de rebalanceo (133 fechas)

**Para Notebook 4:**
- `prices_monthly_open.parquet`: Precios OPEN fin de mes (146 meses × 845 tickers)

**Para Notebook 5:**
- `spy_prices_monthly.parquet`: SPY precios mensuales (133 fechas)
- `spy_log_returns_monthly.parquet`: SPY retornos logarítmicos mensuales (133 fechas)
- `ticker_metadata.csv`: Metadata completa de tickers (845 tickers × 6 columnas)
- `log_returns_monthly.parquet`: Retornos logarítmicos mensuales (146 meses × 845 tickers)

**Estadísticas del Procesamiento:**
- Total tickers: 845
- Elegibles alguna vez: 725 (85.8%)
- Delisted: 195 (23.1%)
- Completitud promedio: 99.84%
- Elegibles por fecha: 497-506 (media: 502.3)

### Dependencias

- **Notebook 1**: Requiere `tickers_data.parquet` y `spy_data.parquet`
- **Librerías**: `numpy`, `pandas`, `matplotlib`, `seaborn`
- **Sin dependencias de Notebooks 3, 4 o 5**: Este notebook prepara todos los datos necesarios

---

## Notebook 3: Implementación de Estrategia Momentum

### Objetivo Principal

Implementación de selección mensual de activos usando metodología MSCI Momentum adaptada. Genera señales de selección para cada fecha de rebalanceo, seleccionando los top 30 activos (20 principales + 10 reservas) basados en scores de momentum compuesto. **OPTIMIZADO**: Usa retornos pre-calculados del Notebook 2, evitando loops de ventanas.

### Flujo de Trabajo Paso a Paso

#### 1. Configuración e Imports
- Define `N_ACTIVOS_SELECCION = 20`: Activos principales en la cartera
- Configura directorios de datos procesados
- Establece estilo de visualizaciones

#### 2. Carga de Datos
- Carga retornos pre-calculados: `returns_12m_rebal`, `returns_6m_rebal`
- Carga `eligibility_mask` y `rebalance_dates`
- Valida que shapes coincidan (formato WIDE consistente)

#### 3. Funciones de Cálculo Vectorizadas

**`calculate_z_scores(returns)`**:
- Calcula z-scores normalizados: `Z = (r_i - μ) / σ`
- Maneja edge cases: pocos activos, sin varianza
- Retorna z-scores con media 0 y desviación estándar 1

**`calculate_momentum_score(R_12, R_6)`**:
- Calcula z-scores de R_12 y R_6
- Alinea índices (solo activos con ambos z-scores)
- Score compuesto: `(Z_12 + Z_6) / 2`
- Balancea momentum largo plazo (12M) y corto plazo (6M)

**`select_top_n(scores, n=30)`**:
- Elimina NaN
- Ordena descendente y toma top n activos
- Retorna lista de tickers ordenados por score

#### 4. Loop Principal de Selección
Para cada fecha de rebalanceo:
1. **Obtener activos elegibles**: Filtra usando `eligibility_mask.loc[fecha]`
2. **Obtener retornos pre-calculados**: R_12 y R_6 solo para elegibles
3. **Calcular scores**: Aplica `calculate_momentum_score()`
4. **Seleccionar top 30**: Usa `select_top_n(scores, n=30)`
   - **Sistema de reservas**: Los primeros 20 son selección principal, siguientes 10 son reservas
5. **Guardar selección**:
   - `ticker_1` a `ticker_20`: Top 20 principales (ordenados por score descendente)
   - `reserva_1` a `reserva_10`: Siguientes 10 mejores candidatos (posiciones 21-30)
6. **Guardar estadísticas**: n_elegibles, n_seleccionados, score_min/max/mean

#### 5. Visualizaciones y Validaciones
- **Gráfico 1**: Evolución de activos elegibles a lo largo del tiempo
- **Gráfico 2**: Evolución de scores de momentum (media, min, max)
- **Análisis de recurrencia**: Tickers más frecuentemente seleccionados
- **Validaciones**:
  - Todas las filas tienen 20 tickers principales: ✓
  - Todas las filas tienen 10 reservas: ✓
  - Sin duplicados en tickers principales: ✓
  - Sin duplicados en reservas: ✓
  - Sin overlap entre principales y reservas: ✓

#### 6. Guardado de Resultados
- Guarda `selecciones_mensuales.csv` con formato:
  - Columna `fecha`
  - 20 columnas `ticker_1` a `ticker_20` (top 20 principales)
  - 10 columnas `reserva_1` a `reserva_10` (reservas para delistings)
- Total: 133 filas × 31 columnas

### Decisiones Técnicas Clave

#### Metodología MSCI Momentum Adaptada
- **Z-scores normalizados dentro del universo elegible**: Se calculan dentro del conjunto de activos elegibles de cada fecha, no sobre todo el universo
- **Justificación**: Normaliza retornos y hace comparables activos con diferentes volatilidades inherentes, esencial para selección justa entre sectores y capitalizaciones
- **Score compuesto**: Media simple de Z_12 y Z_6 balancea momentum largo plazo (tendencias sostenidas) y corto plazo (momentum reciente)

#### Sistema de Reservas (Top 30 en lugar de Top 20)
- **Justificación**: Maneja delistings que ocurren entre fecha de selección (t-1) y fecha de ejecución (t)
- **Implementación**: Selecciona top 30, guarda primeros 20 como principales y siguientes 10 como reservas
- **Beneficio**: Permite reemplazo automático en Notebook 4, garantizando siempre 20 activos válidos en la cartera

#### Uso de Retornos Pre-calculados
- **Optimización crítica**: Usa `returns_12m_rebal` y `returns_6m_rebal` del Notebook 2
- **Ventajas**: Elimina loops costosos de ventanas móviles, reduce tiempo de ejecución en órdenes de magnitud
- **Permite**: Análisis iterativos rápidos, análisis de sensibilidad, validación cruzada

#### Formato de Salida Estructurado
- **CSV en lugar de Parquet**: Archivo pequeño (133 filas), legible por humanos, compatible con motores de backtesting
- **Estructura**: Fecha + 20 tickers principales + 10 reservas
- **Ordenamiento**: Tickers ordenados por score descendente (ticker_1 tiene mayor momentum)

#### Validaciones Exhaustivas
- **Sin duplicados**: Valida que no hay tickers repetidos en principales ni en reservas
- **Sin overlap**: Valida que ningún ticker aparece tanto en principales como en reservas
- **Completitud**: Valida que todas las filas tienen exactamente 20 principales y 10 reservas

### Inputs y Outputs

**Inputs:**
- `returns_12m_rebal.parquet`: Retornos acumulados 12M pre-calculados (desde Notebook 2)
- `returns_6m_rebal.parquet`: Retornos acumulados 6M pre-calculados (desde Notebook 2)
- `eligibility_mask.parquet`: Máscara de elegibilidad (desde Notebook 2)
- `rebalance_dates.csv`: Calendario de rebalanceo (desde Notebook 2)

**Outputs:**
- `selecciones_mensuales.csv`: Selecciones mensuales (133 filas × 31 columnas)
  - Formato: `fecha` + `ticker_1` a `ticker_20` + `reserva_1` a `reserva_10`

**Estadísticas:**
- Tickers únicos seleccionados: 413
- Top 10 más frecuentes: NVDA (76 veces), NFLX (43), AMD (42), MU (33), AVGO (32), etc.

### Dependencias

- **Notebook 2**: Requiere todos los archivos procesados (retornos pre-calculados, eligibility mask, fechas de rebalanceo)
- **Librerías**: `numpy`, `pandas`, `matplotlib`, `seaborn`
- **Sin dependencias de Notebooks 4 o 5**: Este notebook genera las selecciones para el backtesting

---

## Notebook 4: Motor de Backtesting

### Objetivo Principal

Motor de backtesting que simula ejecución real con rebalanceo inteligente mensual, costes transaccionales realistas y manejo de delistings. Conecta las selecciones del Notebook 3 con la ejecución realista del backtesting, manteniendo equiponderación estricta (5% por activo) mientras minimiza costos transaccionales.

### Flujo de Trabajo Paso a Paso

#### 1. Configuración e Imports
- **Parámetros críticos**:
  - `CAPITAL_INICIAL = $250,000`: Capital inicial estándar para estrategias institucionales
  - `TASA_COMISION = 0.23%`: Tasa de comisión realista basada en brokers institucionales
  - `COMISION_MINIMA = $23`: Comisión mínima por trade
  - `N_ACTIVOS = 20`: Activos en la cartera
  - `PESO_POR_ACTIVO = 5%`: Equiponderación estricta

#### 2. Carga de Datos
- Carga `selecciones_mensuales.csv` (incluye 20 principales + 10 reservas)
- Carga `prices_monthly_open.parquet` y `prices_monthly_close.parquet`
- Valida presencia de columnas de reservas
- Valida shapes consistentes y fechas alineadas

#### 3. Clase Portfolio

**Métodos principales:**

**`__init__(capital_inicial)`**:
- Inicializa portfolio con cash inicial y diccionario de posiciones vacío

**`liquidar_selectivo(tickers_a_vender, precios_open, fecha, prices_close_hist)`**:
- Vende solo tickers especificados a precio OPEN
- Maneja delistings: busca último precio disponible si no hay precio OPEN
- Aplica comisión y actualiza cash
- Retorna lista de trades de venta ejecutados

**`ajustar_posicion(ticker, capital_total, precio_close, fecha)`**:
- Ajusta posición existente al 5% del capital total
- **Umbral mínimo de $250**: Solo ajusta si diferencia ≥ $250
  - **Justificación**: Evita sobreajustes menores que generan comisiones desproporcionadas
  - **Rango permitido**: 4.90% - 5.10% del capital total (±0.10%)
- Calcula shares adicionales o a vender según diferencia
- Aplica comisión y actualiza posiciones
- Retorna trade ejecutado o None si no hay ajuste necesario

**`rebalancear_equiponderado(tickers_nuevos, precios_close, fecha)`**:
- **Rebalanceo inteligente**:
  1. Identifica tickers a mantener (intersección de actuales y nuevos)
  2. Ajusta posiciones de tickers que se mantienen (solo si diferencia ≥ $250)
  3. Compra tickers nuevos que no estaban en la cartera
- Calcula capital total (cash + valor posiciones)
- Considera comisiones al calcular shares a comprar
- Maneja casos de cash insuficiente
- Retorna lista de trades ejecutados (ajustes + compras)

**`valor_total(precios_close)`**:
- Calcula valor total del portfolio (cash + posiciones)

**`_buscar_ultimo_precio(ticker, fecha_actual, prices_close_hist)`**:
- Busca último precio disponible antes de fecha_actual
- Usado para manejar delistings en ventas

#### 4. Loop Principal de Backtesting

**Función `reemplazar_tickers_sin_precio()`**:
- **Sistema de reemplazo automático con reservas**:
  - Verifica qué tickers del top 20 no tienen precio CLOSE disponible
  - Para cada ticker faltante, busca en orden (reserva_1 a reserva_10) el primer ticker que SÍ tenga precio válido
  - Evita usar la misma reserva dos veces
  - Retorna lista de 20 tickers válidos y lista de reemplazos realizados
- **Justificación**: Maneja delistings que ocurren entre fecha de selección (t-1) y fecha de ejecución (t)

**Loop por fechas de rebalanceo**:
1. **Extraer tickers principales y reservas** de la selección del mes
2. **Reemplazar tickers sin precio** con reservas válidas
3. **Filtrar tickers** que finalmente no tienen precio (después de intentar reemplazo)
4. **Identificar tickers a vender**: Diferencia entre cartera actual y nueva selección
5. **Vender tickers que salen** (a precio OPEN): `liquidar_selectivo()`
6. **Rebalancear cartera** (a precio CLOSE): `rebalancear_equiponderado()`
   - Ajusta existentes + compra nuevos
7. **Calcular equity** al cierre del día
8. **Validar cartera**: Verifica que tiene exactamente 20 tickers correctos
9. **Guardar snapshot**: equity, cash, número de posiciones

**Resultados**:
- `historial_trades`: Lista completa de todos los trades ejecutados
- `historial_equity`: Snapshots mensuales del portfolio
- `total_reemplazos`: Contador de reemplazos por delistings

#### 5. Visualizaciones del Backtesting
- **Gráfico 1**: Equity Curve del backtesting
- **Gráfico 2**: Retornos mensuales (barras verdes/rojas)
- **Gráfico 3**: Distribución de trades por tipo (VENTA, COMPRA, AJUSTE_COMPRA, AJUSTE_VENTA)
- **Gráfico 4**: Evolución de comisiones acumuladas
- **Gráfico 5**: Trades por fecha de rebalanceo
- **Gráfico 6**: Distribución de trades por mes
- **Comparación**: Rebalanceo inteligente vs Full rebalance (reducción de ~36.6%)

#### 6. Guardado de Resultados
- Guarda `equity_curve.parquet`: Equity curve con fechas, equity, cash, n_positions
- Guarda `trades.csv`: Todos los trades con fecha, ticker, tipo, shares, precio, valor, comisión
- Genera resumen final con estadísticas de eficiencia

### Decisiones Técnicas Clave

#### Rebalanceo Inteligente vs Full Rebalance
- **Estrategia**: Solo ejecuta trades necesarios (vende salidas, ajusta existentes, compra nuevos)
- **Ventaja**: Reduce trades en 30-50% comparado con full rebalance
- **Impacto**: Reduce significativamente costos transaccionales, mejorando retorno neto
- **Implementación**: Mantiene activos existentes, solo ajusta si diferencia ≥ $250

#### Timing de Ejecución Realista
- **VENTA a precio OPEN**: Ventas matutinas liberan capital
- **COMPRA/AJUSTE a precio CLOSE**: Compras al cierre del mismo día
- **Justificación**: Simula ejecución real donde ventas matutinas financian compras del cierre
- **Nota**: En análisis real, sería óptimo considerar curva de volumen diario

#### Umbral Mínimo de Ajuste: $250
- **Optimización**: Aumentado desde $100 a $250 tras análisis
- **Justificación**: Ajustes menores (ej: $100) representan solo 0.04 puntos porcentuales de desviación (5.00% vs 5.04%), lo cual no justifica pagar $23 de comisión mínima
- **Rango permitido**: ±$250 por activo = ±0.10% del capital total (4.90% - 5.10%)
- **Balance**: Mantiene equiponderación estricta mientras reduce sobreajustes innecesarios

#### Sistema de Reemplazo Automático con Reservas
- **Problema**: Delistings entre fecha de selección (t-1) y ejecución (t)
- **Solución**: Reemplazo automático usando reservas del Notebook 3
- **Implementación**: Función `reemplazar_tickers_sin_precio()` busca en orden (reserva_1 a reserva_10) el primer ticker válido
- **Beneficio**: Garantiza siempre 20 activos válidos cuando sea posible

#### Estructura de Costes Realista
- **Comisión**: `max(0.23% × valor, $23)`
- **Justificación**: Refleja costos reales de brokers institucionales
- **Impacto**: Especialmente relevante para trades pequeños donde comisión mínima puede exceder el porcentaje
- **Aplicación**: Se aplica a todas las transacciones (ventas, compras, ajustes)

#### Manejo de Delistings
- **En ventas**: Busca último precio disponible si no hay precio OPEN
- **En compras**: Omite ticker si no hay precio CLOSE (ya manejado por sistema de reservas)
- **En cartera existente**: Si activo en cartera deja de tener precio, se vende a último precio disponible

### Inputs y Outputs

**Inputs:**
- `selecciones_mensuales.csv`: Selecciones mensuales con 20 principales + 10 reservas (desde Notebook 3)
- `prices_monthly_open.parquet`: Precios OPEN mensuales (desde Notebook 2)
- `prices_monthly_close.parquet`: Precios CLOSE mensuales (desde Notebook 2)

**Outputs:**
- `equity_curve.parquet`: Equity curve con fechas, equity, cash, n_positions (133 filas)
- `trades.csv`: Todos los trades ejecutados (3,374 trades con 7 columnas)

**Resultados del Backtesting:**
- Capital inicial: $250,000
- Capital final: $780,904
- Retorno total: +212.36%
- Total trades: 3,374
  - Ventas: 969
  - Compras: 989
  - Ajustes compra: 721
  - Ajustes venta: 695
- Total comisiones: $122,133.01 (48.85% del capital inicial)
- Trades promedio por mes: 25.4 (vs. ~40 con full rebalance)
- Reducción vs full rebalance: 36.6%
- Total reemplazos por delistings: 10

### Dependencias

- **Notebook 2**: Requiere precios OPEN/CLOSE mensuales
- **Notebook 3**: Requiere selecciones mensuales con sistema de reservas
- **Librerías**: `numpy`, `pandas`
- **Sin dependencias de Notebook 5**: Este notebook genera los resultados para el análisis

---

## Notebook 5: Comparativa y Métricas

### Objetivo Principal

Análisis completo de resultados del backtesting, comparación con benchmarks (SPY y Monte Carlo), cálculo de métricas financieras y análisis crítico de la estrategia. Evalúa el desempeño de la estrategia momentum y valida su robustez mediante test de Monte Carlo vectorizado.

### Flujo de Trabajo Paso a Paso

#### 1. Configuración y Carga de Resultados
- Carga `equity_curve.parquet` y `trades.csv` del Notebook 4
- Convierte fecha a índice para análisis temporal
- Calcula retornos mensuales a partir de equity curve
- Muestra resumen preliminar (capital inicial/final, retorno total)

#### 2. Carga del Benchmark SPY
- Carga `spy_prices_monthly.parquet` y `spy_log_returns_monthly.parquet` del Notebook 2
- Alinea con fechas de rebalanceo de la estrategia
- Construye equity curve de SPY con mismo capital inicial
- Calcula retornos mensuales de SPY

#### 3. Cálculo de Métricas Financieras

**Función `calcular_metricas()`** calcula:

1. **CAGR (Compound Annual Growth Rate)**:
   - Fórmula: `((1 + total_return) ^ (1/años) - 1) × 100`
   - Años calculados desde primera a última fecha

2. **Volatilidad (anualizada)**:
   - Fórmula: `std(retornos_mensuales) × √12 × 100`
   - Anualiza multiplicando por √12 para retornos mensuales

3. **Ratio Sharpe (anualizado)**:
   - Fórmula: `(mean(excess_returns) / std(retornos)) × √12`
   - Excess returns: retornos - tasa libre de riesgo mensual (2% anual / 12)
   - Usa tasa libre de riesgo = 2% anual

4. **Ratio Sortino**:
   - Similar a Sharpe pero solo penaliza volatilidad negativa
   - Fórmula: `mean(excess_returns) / downside_std × √12`
   - Downside std: desviación estándar solo de retornos negativos

5. **Máximo Drawdown**:
   - Calcula drawdown como: `(cumulative - running_max) / running_max`
   - Máximo drawdown: mínimo de la serie de drawdowns

6. **Beta y Alpha**:
   - **Beta**: `covarianza(algo, benchmark) / varianza(benchmark)`
   - Mide exposición sistemática al mercado
   - **Alpha**: `retorno_actual - retorno_esperado_CAPM`
   - Retorno esperado CAPM: `risk_free_rate + beta × (benchmark_return - risk_free_rate)`
   - Mide retorno exceso ajustado por riesgo

**Resultados**:
- Crea tabla comparativa: Estrategia vs SPY

#### 4. Visualizaciones

1. **Evolución de Rentabilidad Acumulada (%)**:
   - Gráfico de línea: Estrategia vs SPY a lo largo del tiempo
   - Muestra outperformance/underperformance relativo

2. **Histograma de Retornos Mensuales**:
   - Distribución de retornos mensuales: Estrategia vs SPY
   - Compara formas de distribución

3. **Scatter Plot Anual**:
   - Retorno anual Estrategia vs Retorno anual SPY
   - Línea de igualdad (y=x) para referencia
   - Analiza correlación y recurrencia

4. **Scatter Plot Trimestral**:
   - Similar al anual pero con frecuencia trimestral
   - Mayor granularidad temporal

#### 5. Test de Monte Carlo Vectorizado

**Configuración**:
- `N_SIMULACIONES = 25,000,000`: 25 millones de simulaciones
- `BATCH_SIZE = 1,000`: Procesamiento en batches para optimizar memoria
- `COSTE_REBALANCEO = 0.46%`: 0.23% venta + 0.23% compra (full rebalance cada mes)

**Metodología**:
- Cada mono (simulación):
  1. Vende TODA su posición (100%) cada mes (full rebalance)
  2. Compra 20 activos ALEATORIOS de los ELEGIBLES ese mes
  3. Usa mismo universo que la estrategia (eligibility_mask)
  4. Paga 0.46% por rebalanceo cada mes

**Implementación Vectorizada**:
- **Pre-cálculo de índices elegibles**: Calcula índices de activos elegibles por mes una sola vez
- **Procesamiento en batches**: Procesa 1,000 simulaciones simultáneamente
- **Vectorización completa**: Para cada mes, genera todas las selecciones aleatorias del batch de una vez
- **Cálculos vectorizados**: Retorno promedio mensual, aplicación de costes, retorno acumulado, CAGR
- **Loop interno necesario**: Se requiere loop por mes para respetar eligibility_mask variable mes a mes
- **Trade-off aceptado**: Preferimos comparación justa (eligibility constraint) aunque sea ligeramente más lento

**Resultados**:
- Calcula percentil de la estrategia en la distribución de monos
- Estadísticas: media, mediana, std, percentiles 95 y 99
- Validaciones: CAGR en rango razonable, sin NaN, sin infinitos

**Visualización**:
- Histograma de distribución de CAGR de 25 millones de portfolios aleatorios
- Línea vertical para CAGR de la estrategia con percentil
- Línea vertical para media de monos

#### 6. Análisis Crítico Final

Responde preguntas clave:

1. **Sesgo de Supervivencia**:
   - El universo S&P 500 tiene sesgo inherente del índice
   - La estrategia no introduce sesgo adicional: usa universo dinámico
   - Empresas delisted se incluyen mientras estuvieron disponibles
   - Impacto: Sesgo inherente del S&P 500, no de la implementación

2. **Look-Ahead Bias**:
   - Validado en Notebook 2: eligibility_mask usa solo datos de t-1, t-7, t-13
   - Retornos de momentum calculados con lag de 1 mes
   - Selecciones del Notebook 3 solo usan información disponible en t-1
   - Impacto: Mínimo, validación explícita implementada

3. **Overfitting**:
   - Parámetros fijos (N=20, R_12, R_6, lag=1) sin optimización ex-post
   - Test de Monte Carlo valida que retornos no son solo suerte
   - Percentil > 50% sugiere skill real, no overfitting
   - Impacto: Bajo si percentil es alto, alto si percentil es bajo

4. **Realismo del Rebalanceo**:
   - Rebalanceo inteligente reduce trades vs full rebalance
   - Timing realista: ventas OPEN, compras CLOSE
   - Comisiones realistas: 0.23% con mínimo $23
   - Manejo de delistings implementado
   - Impacto: Positivo, simula ejecución real

5. **Costes de Comisiones**:
   - Total pagado: $122,133.01
   - Como % del capital inicial: 48.85%
   - Número de operaciones: 3,374
   - Comisión promedio: ~$36.22 por operación

### Decisiones Técnicas Clave

#### Métricas Financieras Estándar
- **CAGR**: Métrica anualizada estándar para comparación
- **Volatilidad anualizada**: Multiplica por √12 para retornos mensuales
- **Sharpe y Sortino**: Usan tasa libre de riesgo 2% anual
- **Beta y Alpha**: Calculados usando regresión contra SPY (CAPM)

#### Test de Monte Carlo con Eligibility Constraint
- **Justificación**: Cada mono selecciona solo de activos elegibles ese mes, asegurando comparación justa
- **Full rebalance**: Los monos siempre venden todo y compran todo cada mes (diferente de estrategia real)
- **Coste de rebalanceo**: 0.46% (0.23% × 2) refleja costos reales de compra y venta completas
- **Vectorización**: Procesamiento en batches de 1,000 para optimizar memoria y velocidad

#### Limitaciones Reconocidas del Monte Carlo
- **Loop interno necesario**: Se requiere loop por mes para respetar eligibility_mask variable
- **Alternativa más rápida rechazada**: Seleccionar de todos los activos sería menos justa
- **Trade-off aceptado**: Preferimos comparación justa aunque sea ligeramente más lento

#### Análisis Crítico Estructurado
- Responde explícitamente a 5 preguntas clave sobre sesgos y realismo
- Documenta impactos y mitigaciones
- Transparencia sobre limitaciones y trade-offs

### Inputs y Outputs

**Inputs:**
- `equity_curve.parquet`: Equity curve del backtesting (desde Notebook 4)
- `trades.csv`: Todos los trades ejecutados (desde Notebook 4)
- `spy_prices_monthly.parquet`: SPY precios mensuales (desde Notebook 2)
- `spy_log_returns_monthly.parquet`: SPY retornos mensuales (desde Notebook 2)
- `log_returns_monthly.parquet`: Retornos logarítmicos mensuales (desde Notebook 2)
- `eligibility_mask.parquet`: Máscara de elegibilidad (desde Notebook 2)
- `rebalance_dates.csv`: Calendario de rebalanceo (desde Notebook 2)

**Outputs:**
- Tabla comparativa de métricas: Estrategia vs SPY
- Visualizaciones: 4 gráficos de análisis
- Resultados Monte Carlo: Distribución de 25 millones de portfolios aleatorios
- Análisis crítico: Respuestas a 5 preguntas clave

**Métricas Calculadas:**
- **Estrategia**: CAGR 10.93%, Volatilidad 20.61%, Sharpe 0.51, Sortino 0.21, Max Drawdown -32.54%, Beta 1.10, Alpha 0.29%
- **SPY**: CAGR 13.88%, Volatilidad 14.90%, Sharpe 0.74, Sortino 0.29, Max Drawdown -25.48%, Beta 1.01, Alpha 1.75%

### Dependencias

- **Notebook 2**: Requiere datos de SPY, retornos mensuales, eligibility mask, fechas de rebalanceo
- **Notebook 4**: Requiere resultados del backtesting (equity curve y trades)
- **Librerías**: `numpy`, `pandas`, `matplotlib`, `seaborn`, `scipy`
- **Sin dependencias de otros notebooks**: Este es el notebook final de análisis

---

## Decisiones Arquitectónicas Clave

### Pipeline Completo de Datos

El proyecto sigue un pipeline secuencial bien definido:

```
Notebook 1 (Carga) 
    ↓
Notebook 2 (Preparación) 
    ↓
Notebook 3 (Estrategia) 
    ↓
Notebook 4 (Backtesting) 
    ↓
Notebook 5 (Análisis)
```

**Flujo de datos:**
- **Notebook 1 → Notebook 2**: Datos raw (tickers_data.parquet, spy_data.parquet)
- **Notebook 2 → Notebook 3**: Datos procesados (retornos pre-calculados, eligibility mask, fechas)
- **Notebook 2 → Notebook 4**: Precios mensuales (OPEN y CLOSE)
- **Notebook 3 → Notebook 4**: Selecciones mensuales (con sistema de reservas)
- **Notebook 4 → Notebook 5**: Resultados del backtesting (equity curve, trades)
- **Notebook 2 → Notebook 5**: Datos de SPY y retornos para comparación

### Optimizaciones Clave Implementadas

#### 1. Formato WIDE (fechas × tickers)
- **Justificación**: Permite operaciones vectorizadas eficientes
- **Impacto**: Reduce tiempo de ejecución en órdenes de magnitud
- **Aplicación**: Todos los DataFrames de precios, retornos y elegibilidad

#### 2. Pre-cálculo de Retornos Acumulados
- **Justificación**: Elimina loops costosos en Notebook 3
- **Implementación**: Ventanas móviles vectorizadas en Notebook 2
- **Impacto**: Notebook 3 ejecuta en segundos en lugar de minutos

#### 3. Rebalanceo Inteligente
- **Justificación**: Reduce trades innecesarios
- **Implementación**: Mantiene activos existentes, solo ajusta si diferencia ≥ $250
- **Impacto**: Reduce trades en 36.6% vs full rebalance, ahorrando comisiones significativas

#### 4. Sistema de Reservas para Delistings
- **Justificación**: Maneja delistings entre selección y ejecución
- **Implementación**: Top 30 activos (20 principales + 10 reservas) en Notebook 3, reemplazo automático en Notebook 4
- **Impacto**: Garantiza siempre 20 activos válidos cuando sea posible

#### 5. Monte Carlo Vectorizado
- **Justificación**: Valida robustez de la estrategia
- **Implementación**: Procesamiento en batches de 1,000, vectorización completa dentro de cada batch
- **Impacto**: 25 millones de simulaciones en tiempo razonable (< 24 horas)

### Manejo de Edge Cases

#### Delistings
- **Problema**: Activos que dejan de cotizar entre selección y ejecución
- **Solución**: Sistema de reservas + reemplazo automático
- **Implementación**: Función `reemplazar_tickers_sin_precio()` en Notebook 4

#### Activos sin Precio en Ventas
- **Problema**: Activo en cartera deja de tener precio OPEN
- **Solución**: Busca último precio disponible en histórico
- **Implementación**: Método `_buscar_ultimo_precio()` en clase Portfolio

#### Cash Insuficiente
- **Problema**: No hay suficiente cash para comprar activo nuevo
- **Solución**: Ajusta shares disponibles según cash, omite si no alcanza para mínimo
- **Implementación**: Lógica en `rebalancear_equiponderado()`

#### Forward Fill Controlado
- **Problema**: Gaps pequeños (holidays, suspensiones temporales) vs delistings
- **Solución**: Forward fill con límite de 5 días, solo dentro de vida del activo
- **Implementación**: `ffill(limit=5)` + máscara post-ffill en Notebook 2

#### Universos Pequeños
- **Problema**: Fechas con pocos activos elegibles
- **Solución**: Validaciones en funciones de cálculo (edge cases)
- **Implementación**: Manejo de casos con < 2 activos, sin varianza, etc.

### Validaciones y Controles de Calidad

#### Look-Ahead Bias
- **Validación**: Test automatizado en Notebook 2
- **Implementación**: `validate_no_lookahead()` verifica que elegibilidad no usa información futura
- **Resultado**: Test PASADO (ninguna violación detectada)

#### Integridad de Datos
- **Validaciones en Notebook 1**: Shapes, tipos de datos, rango de fechas
- **Validaciones en Notebook 2**: Formato WIDE, alineación temporal, look-ahead bias
- **Validaciones en Notebook 3**: Sin duplicados, sin overlap, completitud (20 principales + 10 reservas)
- **Validaciones en Notebook 4**: Cartera tiene exactamente 20 tickers después de cada rebalanceo
- **Validaciones en Notebook 5**: CAGR en rango razonable, sin NaN, sin infinitos en Monte Carlo

#### Consistencia Temporal
- **Alineación**: Todos los DataFrames alineados con `rebalance_dates`
- **Lag aplicado**: Retornos de momentum calculados con lag de 1 mes
- **Validación**: R_12 usa retornos desde t-13 a t-1, NO usa mes actual

### Trade-offs Aceptados y Justificados

#### 1. Loop Interno en Monte Carlo
- **Trade-off**: Loop por mes necesario para respetar eligibility_mask variable
- **Justificación**: Comparación justa (eligibility constraint) es más importante que velocidad máxima
- **Alternativa rechazada**: Seleccionar de todos los activos sería más rápido pero menos justo

#### 2. Umbral de Ajuste $250
- **Trade-off**: Permite desviación de ±0.10% del capital total (4.90% - 5.10%)
- **Justificación**: Balance entre equiponderación estricta y costos transaccionales
- **Optimización**: Aumentado desde $100 tras análisis de costos desproporcionados

#### 3. Forward Fill Limitado a 5 Días
- **Trade-off**: Algunos gaps legítimos pueden quedar sin rellenar
- **Justificación**: Evita extender artificialmente vida de activos delisted
- **Alternativa rechazada**: Límite mayor introduciría sesgo de supervivencia

#### 4. Formato CSV para Selecciones
- **Trade-off**: Menos eficiente que Parquet para archivos grandes
- **Justificación**: Archivo pequeño (133 filas), legible por humanos, compatible con motores de backtesting
- **Alternativa rechazada**: Parquet no es crítico para archivo tan pequeño

#### 5. Full Rebalance en Monte Carlo
- **Trade-off**: Monos hacen full rebalance (diferente de estrategia real con rebalanceo inteligente)
- **Justificación**: Simplifica implementación y mantiene comparación justa (mismo universo)
- **Nota**: Esto hace que la comparación sea conservadora (monos pagan más comisiones)

### Conclusiones del Proyecto

El proyecto implementa un pipeline completo de backtesting para una estrategia momentum institucional, con las siguientes características destacadas:

1. **Robustez**: Sistema de reservas maneja delistings, validaciones exhaustivas previenen errores
2. **Eficiencia**: Optimizaciones vectorizadas reducen tiempo de ejecución significativamente
3. **Realismo**: Costes transaccionales realistas, timing de ejecución apropiado, manejo de edge cases
4. **Transparencia**: Análisis crítico explícito de sesgos, limitaciones y trade-offs
5. **Validación**: Test de Monte Carlo con 25 millones de simulaciones valida robustez

El pipeline está diseñado para ser mantenible, extensible y reproducible, con decisiones técnicas bien documentadas y justificadas en cada paso.
