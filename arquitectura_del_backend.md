# Arquitectura del Backend

Este documento especifica la estructura del código, la separación de responsabilidades y el flujo de datos del backend para el **Screener y Análisis Fundamental Cuantitativo (v1.0)**.

## Estructura del Proyecto

```
backend/
├── app/
│   ├── __init__.py
│   ├── config.py               # Configuración global, universo de tickers y constantes
│   ├── data/
│   │   ├── __init__.py
│   │   └── fetcher.py          # Extracción y limpieza de balances contables (yfinance)
│   ├── metrics/
│   │   ├── __init__.py
│   │   └── calculator.py       # Motor matemático de los 4 Pilares fundamentales
│   ├── screener/
│   │   ├── __init__.py
│   │   └── ranker.py           # Factor Scoring y ordenamiento cuantitativo
│   └── utils/
│       ├── __init__.py
│       └── exporter.py         # Persistencia y generación de reportes CSV
├── tests/                      # Suite global de pruebas unitarias y de validación
│   ├── __init__.py
│   ├── test_data_fetcher.py    # Tests de validación de esquemas y NaNs
│   ├── test_metrics_calculator.py # Tests unitarios de fórmulas (DuPont, FCF, ratios)
│   └── test_screener_ranker.py # Tests de ranking y reglas booleanas
├── output/                     # Directorio de salida para reportes (.csv)
│   └── .gitkeep
├── .env.example
├── .gitignore
├── main.py                     # Script principal de ejecución (Orquestador)
├── Makefile                    # Automatización de tareas (test, run-screener, clean)
├── README.md                   # Instrucciones de instalación y uso
└── requirements.txt            # Dependencias del proyecto (pandas, yfinance, pytest)

```

## Flujo de Ejecución del Sistema

El pipeline del backend opera como un proceso secuencial determinista sin estado persistente entre corridas:

```
                     [ main.py (Orquestador) ]
                                │
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ 1. Ingesta y Limpieza (app/data/fetcher.py)                 │
 │    • Consulta API yfinance para universo de tickers          │
 │    • Filtra registros nulos (NaN) o balances corruptos      │
 └──────────────────────────────┬──────────────────────────────┘
                                │ DataFrames limpios
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ 2. Motor de Métricas (app/metrics/calculator.py)             │
 │    • Calcula Rentabilidad: ROE (DuPont), ROIC, Márgenes     │
 │    • Calcula Salud Financiera: Deuda Neta/EBITDA, Current R.│
 │    • Calcula Flujo de Caja: Free Cash Flow (FCF)             │
 │    • Calcula Valuación Relativa: P/E, EV/EBITDA             │
 └──────────────────────────────┬──────────────────────────────┘
                                │ Matriz de métricas calculadas
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ 3. Screener & Scoring (app/screener/ranker.py)              │
 │    • Normaliza métricas mediante Z-Scores                    │
 │    • Aplica ponderación de factores por pilar               │
 │    • Filtra activos con patrimonio negativo o insolvencia   │
 └──────────────────────────────┬──────────────────────────────┘
                                │ Ranking de activos ordenado
                                ▼
 ┌─────────────────────────────────────────────────────────────┐
 │ 4. Exportación (app/utils/exporter.py)                      │
 │    • Escribe tabla consolidada en backend/output/reporte.csv│
 └──────────────────────────────┴──────────────────────────────┘

```

## Componentes y Capas del Backend

### 1. Capa de Datos (`app/data/fetcher.py`)

Responsable exclusiva de la comunicación con fuentes externas y de garantizar la integridad inicial del conjunto de datos.

* **Responsabilidades:**

  * Conectarse a Yahoo Finance mediante la librería `yfinance`.

  * Extraer el Estado de Resultados (*Income Statement*), Balance General (*Balance Sheet*) y Estado de Flujo de Efectivo (*Cash Flow Statement*).

  * Manejar errores de red o tickers no encontrados sin detener la ejecución global.

  * Validar que los campos contables indispensables existan antes de enviar el paquete a la capa analítica.

### 2. Capa de Métricas (`app/metrics/calculator.py`)

Implementa las ecuaciones de finanzas cuantitativas pura sin dependencias externas de almacenamiento o red.

* **Ecuaciones implementadas:**

  * **Rentabilidad y Eficiencia:**
    

    $$
    \text{ROE} = \frac{\text{Net Income}}{\text{Shareholders' Equity}}
    $$

    $$
    \text{ROIC} = \frac{\text{EBIT} \times (1 - t)}{\text{Equity} + \text{Deuda Financiera} - \text{Caja}}
    $$

    $$
    \text{Margen Operativo} = \frac{\text{EBIT}}{\text{Ventas}}, \quad \text{Margen Neto} = \frac{\text{Net Income}}{\text{Ventas}}
    $$

  * **Salud Financiera:**
    

    $$
    \text{Deuda Neta / EBITDA} = \frac{\text{Deuda Total} - \text{Caja y Equivalentes}}{\text{EBITDA}}
    $$

    $$
    \text{Current Ratio} = \frac{\text{Activo Corriente}}{\text{Pasivo Corriente}}
    $$

  * **Flujo de Caja:**
    

    $$
    \text{FCF} = \text{Operating Cash Flow} - \text{CAPEX}
    $$

  * **Valuación:**
    

    $$
    \text{P/E} = \frac{\text{Precio por Acción}}{\text{EPS}}, \quad \text{EV/EBITDA} = \frac{\text{Enterprise Value}}{\text{EBITDA}}
    $$

### 3. Capa de Screening y Ranking (`app/screener/ranker.py`)

Recibe los DataFrames procesados por la capa de métricas y aplica el algoritmo de ordenamiento cuantitativo (*Factor Scoring*).

* **Algoritmo de Scoring:**

  1. **Estandarización:** Convierte cada indicador a un puntaje estadístico $Z = \frac{x - \mu}{\sigma}$.

  2. **Direccionamiento:**

     * Factores positivos (ROE, ROIC, FCF, Márgenes): A mayor valor, mayor score.

     * Factores inversos (Deuda Neta/EBITDA, P/E, EV/EBITDA): A menor valor positivo, mayor score.

  3. **Score Ponderado:** Asigna pesos a cada pilar (Rentabilidad: 35%, Salud Financiera: 25%, Flujo de Caja: 20%, Valuación: 20%).

  4. **Filtros de Exclusión:** Descarta activos con $\text{Equity} \le 0$, $\text{FCF} < 0$ o $\text{Deuda Neta / EBITDA} > 4.5\text{x}$.

### 4. Capa de Utilidades e Exportación (`app/utils/exporter.py`)

Formatea la tabla de resultados y la escribe en disco.

* **Salida predeterminada:** Genera un archivo CSV con fecha en `output/screener_fundamental_YYYYMMDD.csv` conteniendo todas las métricas desglosadas por activo más la columna `Factor_Score`.

## Estrategia de Testing

Las pruebas están centralizadas en la carpeta `tests/` y se ejecutan con `pytest`.

* **`test_data_fetcher.py`:** Utiliza datos simulados (*mocking*) para validar que el sistema responda correctamente ante la falta de filas, valores nulos (`NaN`) o desconexión a la API.

* **`test_metrics_calculator.py`:** Verifica matemáticamente cada fórmula financiera con datasets prefijados donde los resultados teóricos son conocidos. Incluye la verificación del desglose DuPont ($\text{Margen Neto} \times \text{Rotación de Activos} \times \text{Apalancamiento} = \text{ROE}$).

* **`test_screener_ranker.py`:** Comprueba que las reglas de exclusión eliminen empresas insolventes y que el ordenamiento de menor a mayor riesgo sea determinista.

## Comandos de Automatización (`Makefile`)

Todas las operaciones principales del backend están expuestas a través del `Makefile` ubicado en la raíz de `backend/`:

| Comando | Operación | 
| ----- | ----- | 
| `make setup` | Crea el entorno virtual `.venv` e instala todas las dependencias. | 
| `make test` | Corre la suite completa de pruebas unitarias (`pytest tests/ -v`). | 
| `make run-screener` | Ejecuta `main.py`, procesando el universo de acciones y generando el reporte CSV. | 
| `make clean` | Remueve archivos temporales de Python (`__pycache__`, `.pytest_cache`, `.coverage`). | 
