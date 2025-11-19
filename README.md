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
