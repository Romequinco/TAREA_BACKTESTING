# Proyecto de Backtesting - Estrategia Momentum

## ENTREGABLES

Los notebooks principales se encuentran en `notebooks/` y deben ejecutarse, si se desea, en orden secuencial (01 a 05), de lo contrario ya se encuentran correctamente ejecutados. Los resultados del backtesting están disponibles en `datos/backtest/` (equity curve y trades), mientras que los datos procesados intermedios se almacenan en `datos/processed/` (selecciones mensuales, retornos pre-calculados, máscara de elegibilidad). El documento de la tarea se encuentra en `docs/Tarea. Diseño de algoritmos y Backtesting Avanzado.pdf`. 

## ARQUITECTURA DEL PROYECTO

El flujo de datos sigue una arquitectura pipeline desde la carga hasta el análisis final. El sistema utiliza formato WIDE (fechas × tickers) para eficiencia computacional, implementa rebalanceo inteligente y mantiene un sistema de reservas para manejar delistings.

```
[01_Carga_Datos] → datos/raw/ (sp500_history.parquet, spy_data.parquet)
         ↓
[02_EDA_Preparacion] → datos/processed/ (formato WIDE: fechas × tickers)
         ↓
[03_Implementacion_Estrategia] → datos/processed/ (selecciones_mensuales.csv: top 20)
         ↓
[04_Ejecucion_Costes] → datos/backtest/ (equity_curve.csv, trades.csv)
         ↓
[05_Comparativa_Metricas] → Métricas, Monte Carlo (25M Monos), visualizaciones, análisis crítico
```

**Características técnicas:** Formato WIDE para operaciones vectorizadas, rebalanceo mensual con lógica de reservas para activos que dejan de cotizar, y cálculo eficiente de momentum con normalización z-score.

## CONTEXTO

Esta práctica implementa una estrategia de inversión basada en momentum para el universo del S&P 500, utilizando una metodología adaptada de MSCI. El sistema selecciona mensualmente los 20 activos con mayor momentum (combinando señales de 6 y 12 meses) y ejecuta un backtesting completo desde 2015 hasta la actualidad, considerando costes de transacción realistas (0.23% por operación). El análisis incluye métricas de rendimiento, test de robustez mediante Monte Carlo con 25 millones de simulaciones, y una evaluación crítica de los resultados obtenidos.
