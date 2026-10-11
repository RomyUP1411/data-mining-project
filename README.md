# Data Hunters — Seguimiento Presupuestal y Avance Físico de Proyectos de Inversión Pública (Perú)

Proyecto del curso **Data Mining** (Universidad del Pacífico, docente Soledad Espezúa Llerena) desarrollado por el equipo **Data Hunters**: Fernando Torres, Romy Tipacti y Arturo Alvarez.

## 1. Problema Aplicado y Contexto de Negocio

El Sistema Nacional de Programación Multianual y Gestión de Inversiones (**Invierte.pe**) asigna anualmente decenas de miles de millones de soles a proyectos de infraestructura pública en el Perú. No obstante, coexisten dos sistemas de información desintegrados en el Ministerio de Economía y Finanzas (MEF):
1. **`Seguimiento_PI` (MEF):** Registra la asignación y ejecución presupuestal/financiera (PIM, devengado, gasto histórico acumulado).
2. **`Proceso_Seleccion` (MEF):** Registra los procesos de contratación, compras públicas y las valorizaciones técnicas periódicas del avance físico de obra en terreno.

**La problemática crítica:** La falta de cruce y conciliación sistemática entre ambas fuentes impide detectar oportunamente **obras públicas con gasto financiero acelerado que no se traduce en avance físico real** (ejemplo: proyectos con más del 80% de ejecución presupuestal pero menos del 30% de avance físico certificado). Este desbalance es la señal precursora de adendas contractuales inflacionarias, sobrecostos presupuestales, arbitrajes y paralización de obras (*"elefantes blancos"*).

**Objetivo del Proyecto:** Construir una matriz analítica homologada a nivel de proyecto individual que permita cuantificar la **Brecha Gasto - Avance** ($\% \text{Gasto} - \% \text{Avance Físico}$), habilitando modelos no supervisados de Minería de Datos (**Clustering K-Means / DBSCAN** y **Detección de Anomalías con Isolation Forest**) para tipificar perfiles de riesgo y alertar tempranamente proyectos vulnerables.

**Usuario Beneficiario:** Equipos de seguimiento de inversiones (MEF, Unidades Ejecutoras), Órganos de Control Institucional (OCI) y auditores de la **Contraloría General de la República del Perú**.

---

## 2. Fuentes de Datos Oficiales

Ambas fuentes provienen del Portal de Datos Abiertos del MEF:

| Archivo Oficial | Descripción del Contenido | Granularidad en Origen | Enlace Oficial |
|---|---|---|---|
| **`Seguimiento_PI.csv`** | Asignación presupuestal y ejecución financiera (PIM, devengado, gasto acumulado) por año fiscal, pliego y meta presupuestal. | Transaccional anual por partida (696,688 filas) | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/2026-Seguimiento-PI.csv) |
| **`Proceso_Seleccion.csv`** | Procesos de contratación, licitaciones, ítems adjudicados y porcentaje de avance físico acumulado de obra reportado mensualmente. | Transaccional por ítem/contrato/mes (5,672,274 filas) | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/Proceso_Selecccion_Diccionario.csv) |

* **Llave de Integración Homologada:** `codigo_proyecto`, generada al homologar `PRODUCTO_PROYECTO` (`Seguimiento_PI`) y `CODIGO_UNICO` (`Proceso_Seleccion`).
* **Unidad de Análisis Definitiva:** **$1\text{ fila} = 1\text{ Proyecto de Inversión Pública (CUI)}$** en fase de ejecución de obra.

---

## 3. Resolución de Granularidad y Delimitación del Universo Analítico

A partir de la retroalimentación docente del Hito 1, se corrigió la excesiva granularidad y dispersión de los datos crudos mediante un proceso metodológico riguroso:

1. **Exclusión de Códigos Genéricos (Ruido Institucional):** Se identificaron y retiraron **197 códigos presupuestales** compartidos por más de 5 entidades independientes (como el código `2001621` *"Estudios de Preinversión"*, usado por 2,384 unidades ejecutoras distintas), que representaban bolsas de gasto corriente sin expediente técnico de obra física individualizable.
2. **Agregación con `groupby("codigo_proyecto")`:**
   * En `Seguimiento_PI`: Se consolidó el gasto histórico acumulado (`gasto_total`), costo de inversión y años de maduración $\rightarrow$ **52,480 proyectos reales individualizados**.
   * En `Proceso_Seleccion`: Se consolidó el último avance físico certificado (`avance_fisico_pct`), total adjudicado y conteo de contratos $\rightarrow$ **46,742 proyectos con contrataciones**.
3. **Cruce Relacional 1 a 1 (`left join` determinístico):**
   * **35,472 proyectos coincidentes (`both`, 67.59%):** Registran asignación presupuestal y procesos de contratación externa.
   * **17,008 proyectos en `left_only` (32.41%):** Proyectos ejecutados por **Administración Directa** (la entidad ejecuta con su propio personal y maquinaria sin contratista externo) o compras directas $< 8$ UIT.
4. **Delimitación del Universo Analítico (1,978 Obras Activas):**
   * Para alimentar los algoritmos de Minería de Datos sin sesgos por imputación artificial, seleccionamos las **1,978 obras de infraestructura física en ejecución activa** que disponen simultáneamente de avance físico reportado (`avance_fisico_pct.notna()`) y gasto devengado real (`costo_inversion > 0`, `gasto_total > 0`).

---

## 4. Estructura del Proyecto y Guía de Exposición (Hito 2)

El notebook [`Entrega_2/Entrega_2_Data_Hunters.ipynb`](Entrega_2/Entrega_2_Data_Hunters.ipynb) y este repositorio están estructurados para seguir exactamente la **rúbrica de exposición de 12 minutos** exigida para el Hito 2:

**1. Cambios desde el Hito 1 (1.5 min):**
* **Ajuste:** Corrección de la granularidad y ruido institucional. Se excluyeron 197 códigos presupuestales genéricos compartidos.
* **Unidad de Análisis Definitiva:** $1\text{ fila} = 1\text{ Proyecto de Inversión Pública (CUI)}$ en ejecución.

**2. Calidad de Datos (2 min):**
* **Duplicados:** Eliminación de **113,081 duplicados exactos** en `Proceso_Seleccion`.
* **Inconsistencias y Atípicos:** Depuración de rangos imposibles ($[0, 100]\%$ en avance físico, costos $\le 0$). Tratamiento de *outliers* comparando regla IQR vs Z-Score, conservando los datos genuinos de megaproyectos.

**3. Integración y Granularidad (3 min):**
* Agregación previa de ambas tablas a nivel de `codigo_proyecto`.
* **Cruce (Merge):** Relacional 1 a 1 (`left join` con `validate='one_to_one'`). 
* **Validación:** Se obtuvieron 35,472 proyectos coincidentes (asignación + contratación externa). Se excluyeron proyectos por Administración Directa (no cruzaban). El universo final se delimitó a **1,978 obras de infraestructura física en ejecución activa** (con devengado real $>0$ y avance físico reportado).

**4. EDA y Transformaciones (2.5 min):**
* **Hallazgo Principal (EDA):** El **47.6% de obras en Gobiernos Locales** exhibe alerta de gasto adelantado ($>15\%$), con una brecha mediana de $+11.9\%$ (frente a un sano $+2.0\%$ en Gobierno Nacional).
* **Transformaciones aplicadas:** 
  * Derivación de variables de negocio: `pct_ejecucion_financiera`, `brecha_gasto_avance`, `tiempo_maduracion_anios`.
  * Discretización ordinal de la escala de inversión (`pd.cut`).
  * Codificación One-Hot de `nivel_gobierno` (`drop_first=True`).
  * Estandarización y Escalamiento aplicados únicamente sobre las variables continuas.

**5. Matriz Analítica (2 min):**
* **Representación Final:** Matriz consolidada de **1,978 filas × 13 columnas**.
* Cada fila representa un proyecto único con sus variables financieras, operativas (avance) y categóricas ya preparadas para ingestar en los modelos. El CUI se mantiene como índice/llave aislada.

**6. Siguiente Paso (1 min):**
* **Hito 3:** Implementación de modelos no supervisados para tipificar riesgos: **Clustering K-Means / DBSCAN** (perfiles de obra) y **Isolation Forest** (detección de obras anómalas / "elefantes blancos"). Validación cruzada con el portal INFOBRAS.

---

## 5. Instrucciones de Reproducibilidad

### Ejecución del Notebook en Jupyter / Google Colab:
1. Clonar el repositorio:
   ```bash
   git clone https://github.com/RomyUP1411/data-mining-project.git
   ```
2. Si los archivos CSV crudos (`Seguimiento_PI.csv` y `Proceso_Seleccion.csv`) se colocan en la misma carpeta del notebook o en la raíz del proyecto, el notebook los procesará de extremo a extremo.
3. Si no se disponen los archivos crudos (que pesan 1.5 GB), el notebook cuenta con salidas pre-renderizadas completas con todos los gráficos, tablas estadísticas y chequeos de integridad para su revisión inmediata.
4. Abrir y ejecutar:
   ```bash
   jupyter notebook Entrega_2/Entrega_2_Data_Hunters.ipynb
   ```

---

## 6. Estado del Proyecto

- [x] **Hito 1:** Planteamiento del problema, exploración inicial y viabilidad.
- [x] **Hito 2:** Preparación de datos, resolución de granularidad ($1\text{ fila} = 1\text{ Proyecto CUI}$), exclusión de códigos genéricos, EDA formal y primera matriz analítica (1,978 obras × 13 variables).
- [ ] **Hito 3:** Modelado de Minería de Datos (Clustering K-Means / DBSCAN y Detección de Anomalías con Isolation Forest).

---

## 7. Equipo de Investigación

**Data Hunters**
* Fernando Torres
* Romy Tipacti
* Arturo Alvarez

**Curso:** Data Mining (2026-II)  
**Docente:** Soledad Espezúa Llerena (s.espezua@up.edu.pe)  
**Institución:** Universidad del Pacífico  
