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

## 5. Hito 3 — Revisión Metodológica e Implementación Inicial

En esta etapa consolidamos la matriz analítica a $N = 2\,067$ proyectos de inversión pública y ejecutamos una triangulación metodológica con algoritmos de aprendizaje no supervisado para segmentar tipologías de proyectos e identificar anomalías de descalce financiero-físico.

### 5.1. Matriz Analítica y Representación de Datos
* **Unidad de análisis:** $1\text{ fila} = 1\text{ Proyecto de Inversión Pública (CUI)}$.
* **Dimensionalidad:** 12 variables continuas y binarias representativas de ejecución operativa (`pct_ejecucion_financiera`, `avance_fisico_pct`, `brecha_gasto_avance`), magnitud (`costo_inversion`, `gasto_total`, `gasto_anual_promedio`), maduración (`tiempo_maduracion_anios`, `n_registros_presupuesto`) y complejidad contractual/institucional (`n_contratos`, `tamano_inversion_ord`, `gob_GOBIERNOS LOCALES`, `gob_GOBIERNOS REGIONALES`).
* **Estandarización:** Escalamiento z-score con `StandardScaler` ($\mu=0, \sigma=1$).

### 5.2. Técnicas de Minería de Datos y Configuraciones Evaluadas
1. **$K$-Means (Particional):**
   * Evaluación de $K \in [2, 10]$ mediante curva de inercia (*Elbow Method*), Silueta y Davies-Bouldin.
   * **Configuración seleccionada:** $K=4$. Aísla la distorsión del megaproyecto Línea 2 del Metro ($N=1$, costo $>$ S/. 12\,000M) y delimita con nitidez obras municipales ($N=924$), medianas nacionales/regionales ($N=809$) y obras complejas con alta fragmentación contractual ($N=333$, promedio 363 contratos).
2. **DBSCAN (Basado en Densidad):**
   * Parámetro $\text{MinPts} = 2 \times d = 24$.
   * Inspección de la rodilla en el gráfico de $k$-distancias ($k=24$) y prueba de $\varepsilon \in \{1.5, 1.8, 2.0, 2.2\}$.
   * **Configuración seleccionada:** $\varepsilon = 1.8$. Descubre 3 macro-clusters institucionales y etiqueta a 174 proyectos (8.4\%) como **ruido estructural**, los cuales corresponden a casos críticos de descalce y sobrecontratación.
3. **Agrupamiento Jerárquico Aglomerativo:**
   * Comparación de criterios de enlace: `ward`, `complete`, `average` y `single`.
   * Los enlaces *single*, *complete* y *average* colapsan por efecto de encadenamiento (*chaining*) concentrando más de 2,050 obras en un solo nodo.
   * El enlace de **Ward ($K=4$)** genera una estructura balanceada ($1\,117$, $544$, $405$ y $1$) concordante con los arquetipos de $K$-Means.
4. **Reducción Dimensional y Análisis Espacial:**
   * Proyección 2D mediante **PCA** para contrastar visualmente la coherencia geométrica de las particiones.
   * Integración cartográfica con **GeoPandas** y cartografía oficial IGN, identificando una fuerte concentración de inversiones en Cusco (289 proyectos), Puno, Áncash y Lima.

### 5.3. Entregables del Hito 3
* **Notebook Oficial Reproducible:** [`Entrega_3/Entrega_3_Data_Hunters.ipynb`](Entrega_3/Entrega_3_Data_Hunters.ipynb) (ejecutado con todas las celdas, salidas, tablas y gráficos embebidos).
* **Informe Académico en Formato IEEE Conference:** [`Entrega_3/Hito3_IEEE_Data_Hunters.pdf`](Entrega_3/Hito3_IEEE_Data_Hunters.pdf).
* **Mapa Interactivo de Outliers (Folium):** [`Entrega_3/mapa_outliers_dbscan.html`](Entrega_3/mapa_outliers_dbscan.html).
* **Datos Analíticos con Clusters:** [`Entrega_3/matriz_analitica_con_clusters.csv`](Entrega_3/matriz_analitica_con_clusters.csv).

---

## 6. Hito 4 — Paper Final, Pipeline Georreferenciado en HTML y Conclusiones

En la entrega final cerramos el proyecto incorporando la retroalimentación metodológica recibida, el pipeline completo con ejemplos por etapa, el mapa interactivo de clusters y outliers en Folium (`mapa_clusters.html`), la evaluación de proyectos nuevos sin reajustar los modelos, la discusión de limitaciones y las conclusiones y recomendaciones para el usuario.

### 6.1. Pipeline de Desarrollo y Resultados Definitivos
1. **Construcción de la matriz analítica georreferenciada:**
   * Integración 1 a 1 por CUI entre `Seguimiento_PI` y `Proceso_Seleccion` del MEF junto con los polígonos regionales del IGN (`ADM1_ES` en proyección métrica UTM 18S `EPSG:32718`), obteniendo $N = 2\,067$ proyectos con 12 atributos estandarizados (`StandardScaler`) y 2 factores no negativos (`NMF1`, `NMF2` con `MinMaxScaler` + `NMF`).
2. **Selección del modelo y perfiles operativos ($K = 3$):**
   * Convergencia de Inercia (codo con caída de $-19.72\%$), **Silhouette máximo ($0.231$)** y **Davies-Bouldin mínimo ($1.191$)** en **$K = 3$ segmentos** tanto en $K$-Means como en Jerárquico Ward ($1\,522 / 544 / 1$):
     * **Segmento 0 ($43.15\%$, $n = 892$):** Obras regionales y nacionales de mayor escala (S/ $74.1$M promedio) y alta complejidad contractual ($229.7$ contratos por obra).
     * **Segmento 1 ($56.80\%$, $n = 1\,174$):** Obras locales municipales compactas (S/ $5.5$M promedio, `silueta_media = 0.341`, $0$ negativos).
     * **Segmento 2 ($0.05\%$, $n = 1$):** Megaproyecto singular aislado (*Línea 2 del Metro de Lima y Callao*, S/ $12\,075.4$M).
3. **Detección de proyectos atípicos con DBSCAN y Mapa HTML en Folium (`mapa_clusters.html`):**
   * **DBSCAN ($\varepsilon = 1.8, \text{MinPts} = 24$)** separa **174 proyectos ($8.42\%$)** como ruido (outliers), donde la tasa de alerta crítica de gasto adelantado alcanza **$37.93\%$** (frente a $11.41\%$ en los grupos densos) con un promedio de $523.4$ contratos por obra.
   * El mapa interactivo [`Entrega_4/mapa_clusters.html`](Entrega_4/mapa_clusters.html) muestra que **Lima** concentra el mayor volumen y presupuesto en ruido ($74$ outliers, S/ $31\,391.7$M), mientras que **Ucayali ($41.38\%$)**, **Pasco ($23.81\%$)** y **Huancavelica ($14.63\%$)** presentan la mayor proporción relativa de su cartera en condición anómala por fraccionamiento contractual.
4. **Regla operativa para proyectos nuevos:**
   * Aplicación de `scaler.transform` (sin reajustar el escalador), asignación con `kmeans_final.predict`, filtro de revisión manual mediante el **percentil 95 de distancia al centroide ($4.24$)** y regla de vecindad a los puntos núcleo (`core_sample_indices_`) en DBSCAN.

### 6.2. Entregables del Hito 4 (`Entrega_4/`)
* **Paper Final en Formato IEEE (PDF):** [`Entrega_4/Hito4_IEEE_Data_Hunters.pdf`](Entrega_4/Hito4_IEEE_Data_Hunters.pdf).
* **Notebook Final Reproducible:** [`Entrega_4/Entrega_4_Data_Hunters.ipynb`](Entrega_4/Entrega_4_Data_Hunters.ipynb) (ejecutado de inicio a fin sin errores, con todas las tablas y figuras del paper embebidas).
* **Mapa HTML Interactivo de Clusters y Outliers (Folium):** [`Entrega_4/mapa_clusters.html`](Entrega_4/mapa_clusters.html).
* **Matriz Analítica de Entrada y Matriz con Clusters:** [`Entrega_4/matriz_analitica_final.csv`](Entrega_4/matriz_analitica_final.csv) y [`Entrega_4/matriz_analitica_con_clusters.csv`](Entrega_4/matriz_analitica_con_clusters.csv).
* **Cartografía Oficial IGN (Shapefiles Regionales):** [`Entrega_4/peru_shapes/`](Entrega_4/peru_shapes/) y archivo comprimido completo en [`Entrega_3/peru_shapes.zip`](Entrega_3/peru_shapes.zip).
* **Dependencias del Repositorio:** [`requirements.txt`](requirements.txt).

### 6.3. Estructura de Archivos del Repositorio
```text
data-mining-project/
├── README.md
├── requirements.txt
├── Semana 1/ ... Semana 3 - Entrega 1/       # Avances iniciales y Hito 1
├── Entrega_2/                                # Hito 2: limpieza, integración y matriz analítica base
│   ├── Entrega_2_Data_Hunters.ipynb
│   └── matriz_analitica_final.csv
├── Entrega_3/                                # Hito 3: implementación inicial de clustering
│   ├── Entrega_3_Data_Hunters.ipynb
│   ├── Hito3_IEEE_Data_Hunters.pdf
│   ├── mapa_outliers_dbscan.html
│   ├── matriz_analitica_con_clusters.csv
│   └── peru_shapes.zip
└── Entrega_4/                                # Hito 4 (Entrega Final Reproducible)
    ├── Entrega_4_Data_Hunters.ipynb          # Notebook completo ejecutado sin errores
    ├── Hito4_IEEE_Data_Hunters.pdf           # Paper final en formato IEEE
    ├── mapa_clusters.html                    # Mapa interactivo Folium (abrir en navegador)
    ├── matriz_analitica_final.csv            # Matriz de entrada (2,067 proyectos × 17 cols)
    ├── matriz_analitica_con_clusters.csv     # Matriz final enriquecida (2,067 × 23 cols)
    └── peru_shapes/                          # Shapefiles oficiales del IGN (ADM1)
```

---

## 7. Instrucciones de Reproducibilidad

### 7.1. Instalación de Dependencias
```bash
pip install -r requirements.txt
```

### 7.2. Ejecución de Notebooks y Visualización del Mapa HTML
* **Para reproducir el Hito 4 (Entrega Final):** utiliza directamente `Entrega_4/matriz_analitica_final.csv` y `Entrega_4/peru_shapes/` (ambos incluidos en el repositorio):
  ```bash
  jupyter notebook Entrega_4/Entrega_4_Data_Hunters.ipynb
  ```
* **Para visualizar el mapa interactivo en navegador:** abrir [`Entrega_4/mapa_clusters.html`](Entrega_4/mapa_clusters.html) en Chrome o Edge (por ejemplo `start Entrega_4/mapa_clusters.html` en Windows).
* **Para reproducir desde fuentes crudas (Hito 2):** descargar `Seguimiento_PI.csv` y `Proceso_Seleccion.csv` desde los enlaces oficiales de la Sección 2 (omitidos en GitHub por tamaño) y ejecutar [`Entrega_2/Entrega_2_Data_Hunters.ipynb`](Entrega_2/Entrega_2_Data_Hunters.ipynb).
* **Para reproducir el Hito 3:**
  ```bash
  jupyter notebook Entrega_3/Entrega_3_Data_Hunters.ipynb
  ```

---

## 8. Estado del Proyecto

- [x] **Hito 1:** Planteamiento del problema, exploración inicial y viabilidad.
- [x] **Hito 2:** Preparación de datos, resolución de granularidad ($1\text{ fila} = 1\text{ Proyecto CUI}$), exclusión de códigos genéricos, EDA formal y primera matriz analítica reproducible.
- [x] **Hito 3:** Revisión metodológica e implementación inicial de minería de datos: Clustering no supervisado (K-Means, DBSCAN, Jerárquico Ward), reducción dimensional con NMF, análisis geoespacial (GeoPandas y Folium) e informe académico formato IEEE.
- [x] **Hito 4:** Paper final IEEE, pipeline de desarrollo con ejemplo por etapa, resultados definitivos ($K=3$ y outliers DBSCAN), mapa interactivo HTML (`mapa_clusters.html`), evaluación de proyectos nuevos, discusión de limitaciones, conclusiones y recomendaciones para el usuario.

---

## 9. Equipo de Investigación

**Data Hunters**
* Fernando Torres
* Romy Tipacti
* Arturo Alvarez

**Curso:** Data Mining (2026-II)  
**Docente:** Soledad Espezúa Llerena (s.espezua@up.edu.pe)  
**Institución:** Universidad del Pacífico  
