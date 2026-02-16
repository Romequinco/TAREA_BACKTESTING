# Proyecto de Backtesting - Estrategia Momentum

Tarea de backtesting y algoritmos grado mIAx

## Estructura del Proyecto

Este proyecto está organizado en 5 notebooks Jupyter que deben ejecutarse en orden:

1. **01_Carga_Datos.ipynb**: Carga y validación de datos históricos
2. **02_EDA_Preparacion.ipynb**: Análisis exploratorio y preparación de datos
3. **03_Implementacion_Estrategia.ipynb**: Implementación de la estrategia Momentum
4. **04_Ejecucion_Costes.ipynb**: Motor de backtesting con costes
5. **05_Comparativa_Metricas.ipynb**: Análisis de resultados y comparación con benchmarks

## Requisitos

### Librerías Permitidas (según PDF)
- numpy
- pandas
- yfinance
- matplotlib
- seaborn
- scipy
- pyarrow

**IMPORTANTE:** El uso de cualquier otra librería limitará la nota máxima a 5.0/10.0

### Datos Necesarios

1. **Datos históricos de precios del S&P 500**
   - Disponible en: https://drive.google.com/file/d/1nvubXdAu0EONlrP_yrURZbnPhBQ-uDaB/view?usp=sharing
   - Debe contener precios desde al menos 2014 (para calcular momentum con lag)
   - Formato esperado: CSV o Parquet con fechas como índice y símbolos como columnas

## Parámetros del Backtesting

- **Capital inicial:** $250,000
- **Período:** 1 enero 2015 - actualidad
- **Universo:** Activos del S&P 500 (últimos 13 meses)
- **Benchmark:** SPY (S&P 500 ETF)
- **Rebalanceo:** Mensual (último día hábil del mes)
- **Estrategia:** Momentum (metodología MSCI adaptada)
- **Costes:** 0.23% por operación, mínimo $23 por orden

## Metodología de la Estrategia

### Paso A: Cálculo de Momentum
- **Momentum 12 meses (R_12):** Retorno logarítmico desde mes t-13 al mes t-1
- **Momentum 6 meses (R_6):** Retorno logarítmico desde mes t-7 al mes t-1
- **Lag:** 1 mes (excluye mes actual para evitar reversión a la media)

### Paso B: Normalización Z-Score
- Normalización dentro del universo de cada mes
- Fórmula: Z = (X - μ) / σ

### Paso C: Selección y Pesos
- **Score final:** Media simple de Z_6 y Z_12
- **Selección:** Top 20 activos por score
- **Pesos:** 5% del capital en cada activo (igual ponderación)

## Reglas de Ejecución

1. **Venta:** Precio OPEN del día de rebalanceo
2. **Compra:** Precio CLOSE del mismo día
3. **Costes:** 0.23% con mínimo $23 por orden
4. **Activos que dejan de cotizar:** Vender al CLOSE, mantener cash hasta siguiente rebalanceo

## Métricas Requeridas

- CAGR (Compound Annual Growth Rate)
- Volatilidad
- Ratio Sharpe
- Ratio Sortino
- Máximo Drawdown
- Beta (vs SPY)
- Alpha (vs SPY)

## Visualizaciones Requeridas

1. Evolución de rentabilidad acumulada (%) vs SPY
2. Histograma de retornos mensuales (algoritmo vs SPY)
3. Scatter plot de retornos anuales (algoritmo vs SPY)
4. Scatter plot de retornos trimestrales (algoritmo vs SPY)

## Test de Monte Carlo

- **Número de carteras:** ≥ 25,000,000
- **Tiempo máximo:** < 24 horas
- **Reglas:** 
  - Coste de rebalanceo: 0.46% (0.23% x 2)
  - Sin mínimo de $23 por orden
  - 20 activos aleatorios con pesos 5% cada uno

## Análisis Crítico Requerido

Responder a las siguientes preguntas:

1. ¿Cómo nos está afectando el sesgo de supervivencia?
2. ¿Cómo hemos garantizado que no tengamos un problema de look-ahead?
3. ¿Crees que existe un problema de overfitting?
4. ¿Hemos realizado un rebalanceo irrealista?
5. ¿Cuánto dinero hemos pagado en comisiones de compraventa?

## Formato de Entrega

1. **5 notebooks separados** (todos ejecutados)
2. **Archivo CSV** con los 20 activos seleccionados por fecha de rebalanceo
3. **Todo comprimido en un único archivo ZIP**

## Criterios de Evaluación

- **Implementación Técnica (4 puntos):** Correcta aplicación de lógica Momentum, z-scores y rebalanceo
- **Motor de Backtesting y Costes (2 puntos):** Rigor en ejecución OPEN/CLOSE y aplicación exacta de costes
- **Análisis de Robustez (2 puntos):** Ejecución correcta de Monte Carlo e interpretación coherente
- **Calidad de Visualización y Reflexión (2 puntos):** Claridad de gráficos y profundidad en análisis

### Bonificaciones Especiales

- **+1 punto:** Arquitectura del software (uso correcto de clases y funciones)
- **+1 punto:** Calidad del código (PEP 8, limpieza visual, comentarios detallados)

## Notas Importantes

1. **Todos los notebooks deben estar ejecutados** antes de la entrega
2. Las celdas no ejecutadas se puntuarán con 0
3. El código debe seguir PEP 8
4. Usar retornos logarítmicos para cálculo de señales
5. Implementación modular con funciones y/o clases

## Estructura de Directorios

```
TAREA_BACKTESTING/
├── notebooks/
│   ├── 01_Carga_Datos.ipynb
│   ├── 02_EDA_Preparacion.ipynb
│   ├── 03_Implementacion_Estrategia.ipynb
│   ├── 04_Ejecucion_Costes.ipynb
│   └── 05_Comparativa_Metricas.ipynb
├── docs/
│   └── Tarea. Diseño de algoritmos y Backtesting Avanzado.pdf
├── data/                    # (se crea al ejecutar)
│   ├── price_data_clean.parquet
│   ├── log_returns_monthly.parquet
│   ├── selected_assets.csv
│   ├── equity_curve.csv
│   └── ...
└── README.md
```

## Instrucciones de Uso

1. Descargar datos históricos desde Google Drive
2. Colocar archivo en directorio `data/` o ajustar ruta en Notebook 1
3. Ejecutar notebooks en orden secuencial
4. Verificar que todas las celdas se ejecuten correctamente
5. Completar análisis crítico en Notebook 5
6. Comprimir todos los notebooks ejecutados en un ZIP

## Contacto y Soporte

Para dudas sobre la implementación, consultar el PDF original de la práctica en la carpeta `docs/` o contactar al profesor.

---

**Última actualización:** 2025-02-16
