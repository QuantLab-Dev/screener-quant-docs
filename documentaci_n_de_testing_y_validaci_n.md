# Documentación de Testing y Validación Cuantitativa
## Framework de Pruebas Unitarias e Integridad de Datos con `pytest`

### Control de Versiones
* **Documento:** Plan de Testing y Auditoría de Modelos
* **Versión:** 1.0.0
* **Herramientas:** Python 3.10+, `pytest`, `pytest-cov`, `unittest.mock`

---

## 1. Filosofía de Testing Cuantitativo

En software financiero, las pruebas unitarias cumplen dos roles fundamentales:
1. **Verificación de Software:** Garantizar que las funciones no arrojen excepciones inesperadas ante datos atípicos.
2. **Validación de Modelos (Model Validation):** Asegurar que las ecuaciones matemáticas de la literatura financiera (como la Descomposición de DuPont o el ROIC) estén correctamente codificadas y no distorsionen los resultados.

---

## 2. Estructura de la Suite de Pruebas

Las pruebas se dividen en tres clases principales dentro del directorio `tests/`:

```text
tests/
├── test_data_loader.py       # Pruebas de ingesta y detección de anomalias en datos
├── test_metrics_calculator.py# Pruebas matematicas sobre los 4 pilares
└── test_screener.py          # Pruebas de filtrado logico y scoring
```

---

## 3. Especificación de Casos de Prueba (Test Cases)

### 3.1 Módulo: Data Loader (`test_data_loader.py`)

#### Test Case DAT-01: Control de Valores Nulos e Infinitos
* **Objetivo:** Verificar que el limpiador de datos convierta cadenas vacías o divisiones por cero en valores flotantes nulos manejables (`np.nan`).
* **Criterio de Aceptación:** Ninguna serie devuelta debe contener strings corruptos o `inf`.

#### Test Case DAT-02: Escudo contra Patrimonio Neto Negativo (Protección de Quiebra)
* **Objetivo:** Si una empresa acumula pérdidas históricas tales que su Patrimonio Neto es negativo ($\text{Equity} < 0$), calcular $\text{ROE} = \frac{\text{Net Income}}{\text{Equity}}$ produciría un número positivo engañoso si las ganancias también son negativas (menos por menos es más).
* **Entrada:** `Net Income = -500`, `Equity = -1000`.
* **Resultado Esperado:** El sistema debe detectar la anomalía, marcar una bandera de riesgo (`Negative_Equity_Flag = True`) y asignar $\text{ROE} = \text{NaN}$.

---

### 3.2 Módulo: Calculadora de Métricas (`test_metrics_calculator.py`)

#### Test Case MET-01: Identidad Matemática de DuPont
* **Objetivo:** Validar la exactitud del ROE mediante el producto de sus tres factores componentes:

  $$\text{ROE} = \text{Margen Neto} \times \text{Rotación de Activos} \times \text{Apalancamiento Financiero}$$

* **Datos de Prueba Mockeados:**
  * Ventas ($\text{Revenue}$) = $\$1,000,000$
  * Beneficio Neto ($\text{Net Income}$) = $\$150,000$
  * Activos Totales ($\text{Total Assets}$) = $\$2,000,000$
  * Patrimonio Neto ($\text{Equity}$) = $\$800,000$
* **Valores Esperados:**
  * $\text{Margen Neto} = \frac{150,000}{1,000,000} = 0.15$ ($15\%$)
  * $\text{Rotación Activos} = \frac{1,000,000}{2,000,000} = 0.5$
  * $\text{Apalancamiento} = \frac{2,000,000}{800,000} = 2.5$
  * $\text{ROE Directo} = \frac{150,000}{800,000} = 0.1875$ ($18.75\%$)
  * $\text{ROE DuPont} = 0.15 \times 0.5 \times 2.5 = 0.1875$ ($18.75\%$)
* **Assert en `pytest`:** `assert pytest.approx(roe_directo) == roe_dupont`

#### Test Case MET-02: Cálculo de Flujo de Caja Libre (FCF)
* **Objetivo:** Verificar la deducción correcta de las inversiones en bienes de capital (CAPEX) sobre el flujo operativo.

  $$\text{FCF} = \text{Operating Cash Flow} - \text{CAPEX}$$

* **Datos de Prueba Mockeados:**
  * $\text{Operating Cash Flow} = \$450,000$
  * $\text{CAPEX} = \$120,000$
* **Resultado Esperado:** $\text{FCF} = \$330,000$.

#### Test Case MET-03: Deuda Neta / EBITDA
* **Objetivo:** Validar la métrica de cobertura de apalancamiento financiero.

  $$\text{Deuda Neta / EBITDA} = \frac{\text{Deuda Total} - \text{Caja y Equivalentes}}{\text{EBITDA}}$$

* **Datos de Prueba Mockeados:**
  * Deuda Total = $\$500,000$
  * Caja = $\$100,000$
  * EBITDA = $\$200,000$
* **Resultado Esperado:** $\frac{500,000 - 100,000}{200,000} = 2.0\text{x}$.

---

### 3.3 Módulo: Screener & Ranking (`test_screener.py`)

#### Test Case SCR-01: Filtrado Booleano Combinado
* **Objetivo:** Evaluar que el screener reduzca correctamente la matriz de acciones según las restricciones impuestas por el usuario.
* **Criterios de Filtro:**
  * $\text{ROE} \ge 0.15$
  * $\text{Deuda Neta / EBITDA} \le 2.5$
* **Resultado Esperado:** Las empresas que incumplan al menos una de las dos condiciones deben ser removidas del DataFrame filtrado.

#### Test Case SCR-02: Normalización por Z-Score
* **Objetivo:** Confirmar que la estandarización de variables para el *Factor Score* mantenga media $\mu = 0$ y desviación estándar $\sigma = 1$:

  $$z_i = \frac{x_i - \mu}{\sigma}$$

---

## 4. Estrategia de Mocking para APIs Externas

Para evitar depender de una conexión a internet activa y evitar que los tests fallen si Yahoo Finance modifica su interfaz temporalmente, se implementa *Mocking* con `unittest.mock`.

### Código de Ejemplo de Test con Mocking (`tests/test_metrics_calculator.py`)

```python
import pytest
import pandas as pd
import numpy as np
from unittest.mock import MagicMock

# Importación hipotética de nuestros módulos
# from src.metrics import FinancialMetricsCalculator

def test_dupont_identity_mocked():
    # 1. Crear dataset sintético contable
    mock_financial_data = pd.DataFrame({
        'Revenue': [1000000.0],
        'NetIncome': [150000.0],
        'TotalAssets': [2000000.0],
        'StockholdersEquity': [800000.0]
    }, index=['KO'])

    # 2. Instanciar la calculadora con los datos simulados
    # calc = FinancialMetricsCalculator(mock_financial_data)
    
    # Simulación de cálculos internos
    net_margin = mock_financial_data['NetIncome'] / mock_financial_data['Revenue']
    asset_turnover = mock_financial_data['Revenue'] / mock_financial_data['TotalAssets']
    equity_multiplier = mock_financial_data['TotalAssets'] / mock_financial_data['StockholdersEquity']
    
    roe_direct = mock_financial_data['NetIncome'] / mock_financial_data['StockholdersEquity']
    roe_dupont = net_margin * asset_turnover * equity_multiplier

    # 3. Validar igualdad matemática
    np.testing.assert_allclose(roe_direct.values, roe_dupont.values, rtol=1e-5)
```

---

## 5. Comando de Ejecución de Pruebas

Para correr toda la batería de pruebas unitarias y verificar la cobertura de código, se ejecutará el comando del `Makefile`:

```bash
make test
```

O directamente en la terminal mediante `pytest`:

```bash
pytest tests/ -v --cov=src --cov-report=term-missing
```