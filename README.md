# Sprint2D – Churn de Clientes Olist (Sprint 1 + Sprint 2 + Pipeline Mensual)

Repositorio del proyecto de **Análisis de Churn** usando el dataset público de Olist (marketplace brasileño de e-commerce).

El trabajo se organiza en dos sprints y un pipeline mensual:

- **Sprint 1:** problema de negocio, EDA, hipótesis, definición de target preliminar y métricas base.
- **Sprint 2:** pipeline reproducible, data cleaning, creación de features, KPI finales, target final, modelos y selección de variables.
- **Pipeline mensual:** simulación de incorporación mensual de nuevos datos mediante una **fecha de corte por período (YYYY-MM)** y persistencia de resultados (snapshots mensuales en Parquet + histórico de KPIs).

Notebook principal:  
`notebooks/Sprint2D_Churn_Olist_DRIVE.ipynb`  
(En el repo puede estar en la raíz; ajustar la ruta según corresponda.)

---

## 1. Problema de negocio y objetivo del MVP

**Contexto – Olist**

- Olist es un marketplace donde múltiples **sellers** venden productos a **clientes finales** en Brasil.
- El negocio quiere **identificar clientes con riesgo de abandono (churn)** para priorizar acciones de retención.

**Definiciones clave**

- **Unidad de análisis:** cliente final (`customer_unique_id`).
- **Churn (definición operativa):** cliente que **compró en el pasado** pero lleva tantos días sin comprar que, desde el punto de vista del negocio, se considera **inactivo/perdido**.
- **Periodicidad:** análisis **mensual**, con cortes a fin de mes.

**Objetivo del MVP**

Construir un pipeline que, para un **período objetivo (YYYY-MM)**:

1. Reciba los datos transaccionales de Olist y los recorte hasta una **fecha de corte** (último día del mes).
2. Genere una **master table por cliente** con decenas de features (RFM, valor económico, logística, reviews, etc.).
3. Defina un **target de churn** (preliminar y final) basado en la **recencia de compra**, con umbrales obtenidos desde los datos (percentiles), sin números “quemados”.
4. Entrene al menos dos modelos de clasificación (baseline + modelo más robusto), priorizando **recall** sobre clientes churn.
5. Calcule **KPIs de negocio** (churn rate, retención, gasto promedio, etc.) y los persista como snapshots mensuales.

---

## 2. Hipótesis de negocio y métricas

Las hipótesis se formulan a partir del conocimiento de negocio y de la EDA inicial sobre el histórico de pedidos.

### 2.1 Hipótesis de negocio (Sprint 1)

1. **Recencia y churn**  
   A mayor número de días desde la última compra del cliente (`recency_days`), **mayor probabilidad de churn**.

2. **Frecuencia y valor económico**  
   Clientes con **baja frecuencia de compra** (`frequency` baja) y **bajo valor económico total** (`order_value_sum` bajo) tienen **mayor riesgo de churn** que quienes compran más seguido y con tickets mayores.

3. **Comportamiento de pago**  
   Clientes que realizan compras con **muchas cuotas** (`avg_installments` alta) y **tickets promedio más bajos** (`order_value_mean` bajo) tienden a mostrar **mayor probabilidad de abandono**.

4. **Experiencia de entrega y satisfacción**  
   Peores experiencias logísticas y de servicio —retrasos de entrega (`delivery_delay_days` positivos, `pct_delayed_orders` alto) y **reviews bajas** (`avg_review_score` < 4)— se esperan **correlacionar positivamente con el churn**.

5. **Ubicación geográfica**  
   Ciertos estados (`customer_state`) podrían presentar **tasas de churn sistemáticamente más altas** por temas logísticos (distancia, tiempos de entrega, disponibilidad de sellers), por lo que la localización del cliente aporta información sobre el riesgo de abandono.

### 2.2 Métricas base

**Métricas técnicas del modelo**

- **Recall** sobre la clase positiva (churn).
- **F1-score**.
- **ROC-AUC**.

**KPIs de negocio**

- **Churn rate mensual**: % de clientes que pasan a estado churn en el período.
- **Retention rate mensual**: % de clientes que se mantienen activos.
- **Gasto promedio por cliente**: ticket promedio de clientes activos vs. churn para estimar el impacto económico de perder clientes.

---

## 3. Dataset utilizado

Se utiliza el dataset público de Olist, normalmente descargado de Kaggle, que incluye (entre otras):

- `olist_orders_dataset.csv`
- `olist_order_items_dataset.csv`
- `olist_order_payments_dataset.csv`
- `olist_order_reviews_dataset.csv`
- `olist_customers_dataset.csv`
- `product_category_name_translation.csv` (opcional)

En el notebook, los archivos se ubican en un directorio tipo:

```text
data_raw/
    olist_orders_dataset.csv
    olist_order_items_dataset.csv
    olist_order_payments_dataset.csv
    olist_order_reviews_dataset.csv
    olist_customers_dataset.csv
    ...
```

> Ajustar la ruta `DATA_DIR` del notebook según la ubicación real de los CSV (local, Colab + Google Drive, etc.).

---

## 4. Estructura del proyecto

La estructura recomendada del repo es:

```text
.
├── data_raw/                  # CSV originales (no se versionan en Git por tamaño)
│   └── ...
├── data_processed/
│   ├── master/                # Master tables por período (orders_enriched_YYYY-MM.parquet)
│   ├── features/              # customer_features_YYYY-MM.parquet
│   └── metrics/               # kpis_churn_history.csv (histórico de KPIs)
├── notebooks/
│   └── Sprint2D_Churn_Olist_DRIVE.ipynb
├── src/                       # (opcional) refactor de funciones del notebook a .py
├── README.md
└── requirements.txt           # (opcional) librerías necesarias
```

---

## 5. Descripción del notebook principal

El notebook `Sprint2D_Churn_Olist_DRIVE.ipynb` integra:

1. **Introducción, problema de negocio y objetivo del MVP**  
   - Definición de churn, unidad de análisis, periodicidad, alcance del MVP.

2. **Configuración del entorno e imports**  
   - Librerías (`pandas`, `numpy`, `matplotlib`, `scikit-learn`, etc.).
   - Configuración de rutas (`DATA_DIR`, `PROCESSED_DIR`).

3. **Montaje de Google Drive (Colab) y rutas de datos** (si aplica).

4. **Carga de datos Olist y tablas base**  
   - Función `load_olist_data()` para leer todos los CSV desde `data_raw/`.
   - Revisiones básicas de tamaño, rangos de fechas y valores faltantes.

5. **EDA inicial (Sprint 1)**  
   - Distribución temporal de pedidos, clientes y estados (`order_status`).
   - Análisis exploratorio de recencia, frecuencia y valores de pedido.

6. **Parámetros del pipeline mensual**  
   - Definición de `FECHA_CORTE` por período objetivo (YYYY-MM).
   - Rango temporal de datos a considerar.

7. **Funciones de pipeline (limpieza, recorte, master de órdenes)**  
   - `clean_orders`, `clean_order_items`, `clean_customers`, etc.  
   - `cut_data_to_fecha_corte(...)` para recortar todas las tablas al mes objetivo.  
   - `build_orders_enriched(...)` para construir una tabla de órdenes enriquecida con pagos, ítems, logística y reviews.

8. **Master table por cliente y Feature Engineering**  
   - `build_customer_features(orders_enriched, fecha_corte)` genera la tabla `customer_features` a nivel `customer_unique_id` con features de:
     - **RFM** (recencia, frecuencia, valor total).  
     - **Valor económico** (sumas, medias, desviaciones de montos).  
     - **Logística** (tiempos de entrega, retrasos, pedidos adelantados).  
     - **Pagos** (cantidad de cuotas, valores promedio).  
     - **Satisfacción** (scores de reviews y cantidades).  
   - Se incluye un bloque de **limpieza avanzada**:
     - Imputación de nulos numéricos (medianas, ceros para contadores/porcentajes).  
     - Winsorización de outliers (p1–p99) en las variables continuas.

9. **Definición de target de churn (preliminar y final) + KPIs**  
   - Análisis de la distribución de **días entre compras** (`days_between_orders`).  
   - Cálculo de percentiles (P90, P99) para definir thresholds **sin números quemados**.  
   - Creación de `churn_prelim` y `churn_final` a partir de la recencia.  
   - Cálculo de KPIs de negocio con `compute_basic_kpis(...)`.

10. **Preparación de datos y selección de variables**  
    - Construcción de la **matriz de características X** (numéricas + dummies de `customer_state`) y la target `y`.  
    - Análisis de correlaciones, pruebas **Chi²** y **ANOVA F-test** para evaluar relevancia de variables.  
    - Construcción de una versión reducida (`X_reduced`) usando las variables más importantes para comparar modelos.

11. **Modelos (Logistic Regression y Random Forest)**  
    - Entrenamiento y evaluación de **Regresión Logística** y **Random Forest**.  
    - Cálculo de **precision, recall, F1-score, ROC-AUC** y matrices de confusión.  
    - Análisis de importancias de variables para interpretar el modelo.

12. **Pipeline mensual `run_monthly_pipeline()` y snapshots**  
    - Función principal:

      ```python
      run_monthly_pipeline("2018-06", DATA_DIR, PROCESSED_DIR)
      ```

      que ejecuta de punta a punta:

      1. Carga y limpieza de datos crudos.
      2. Recorte al período objetivo.
      3. Construcción de `orders_enriched`.
      4. Construcción y limpieza de `customer_features`.
      5. Cálculo de target y KPIs.
      6. Guardado de outputs:
         - `data_processed/master/orders_enriched_YYYY-MM.parquet`
         - `data_processed/features/customer_features_YYYY-MM.parquet`
         - `data_processed/metrics/kpis_churn_history.csv` (append).

    - El notebook incluye ejemplos ejecutando el pipeline para varios meses de 2018.

13. **Notas para Git y storytelling**  
    - Recomendaciones sobre qué partes mostrar en la presentación y cómo contar la historia del proyecto.

---

## 6. Cómo ejecutar el notebook / pipeline

1. **Clonar el repositorio**

   ```bash
   git clone https://github.com/<usuario>/<repo>.git
   cd <repo>
   ```

2. **Colocar los datos crudos**

   - Descargar los CSV originales de Olist.
   - Guardarlos en `data_raw/` (o ajustar `DATA_DIR` en el notebook).

3. **Instalar dependencias** (opcional, si se usa entorno local)

   ```bash
   pip install -r requirements.txt
   ```

   En Colab, las librerías principales ya suelen venir instaladas.

4. **Abrir el notebook**

   - Local: con Jupyter / VSCode.  
   - Colab: subir el `.ipynb` o abrirlo desde GitHub.

5. **Configurar rutas**

   - Revisar la sección de configuración del notebook y ajustar:
     - `DATA_DIR` → carpeta donde están los CSV.
     - `PROCESSED_DIR` → carpeta para guardar `master/`, `features/` y `metrics/`.

6. **Ejecutar las celdas en orden**

   - Desde la introducción hasta la definición de modelos.
   - Para correr el pipeline para un mes específico:

     ```python
     run_monthly_pipeline("2018-06", DATA_DIR, PROCESSED_DIR)
     ```

   - Verificar que se crean los archivos Parquet y el histórico `kpis_churn_history.csv`.

---

## 7. Limitaciones y posibles mejoras

- Incorporar más features de comportamiento (por ejemplo, tipos de productos, diversidad de categorías, concentración en pocos sellers, etc.).
- Probar más modelos (XGBoost, LightGBM) y técnicas de calibración de probas.
- Agregar un **dashboard interactivo** (Streamlit/Power BI) usando los snapshots generados.
- Conectar el pipeline a una fuente real de datos (base de datos OLTP u orquestador tipo Airflow).

---

## 8. Créditos

Proyecto desarrollado como parte de la materia de **Modelamiento de Datos / Inteligencia Artificial aplicada**, usando el dataset público de Olist.
