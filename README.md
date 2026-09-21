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

## 4. Estructura del Código Fuente (Hito 2)

El notebook [`Entrega_2/Entrega_2_Data_Hunters.ipynb`](Entrega_2/Entrega_2_Data_Hunters.ipynb) desarrolla sistemáticamente los componentes de la entrega:

1. **Marco de Trabajo y Reformulación:** Justificación de negocio, corrección estructural de granularidad y cascada de delimitación del universo de modelado.
2. **Carga e Inspección de Fuentes:** Diagnóstico inicial y detección flexible de rutas en entorno local o Colab.
3. **Validación de Claves y Duplicados:** Eliminación de **113,081 duplicados exactos** en `Proceso_Seleccion` y exclusión de los 197 códigos genéricos.
4. **Diagnóstico y Calidad de Datos:** Depuración de rangos inválidos ($[0, 100]\%$ en avance, costos $\le 0$) y análisis comparativo de atípicos (regla IQR vs. Z-Score).
5. **Integración Relacional 1 a 1:** Cruce con validación estricta (`validate='one_to_one'`) y verificación de consistencia en registros emparejados y no emparejados.
6. **Análisis Exploratorio de Datos (EDA):** 4 gráficos clave interpretados bajo la estructura pedagógica UP (*Observación $\rightarrow$ Evidencia $\rightarrow$ Interpretación $\rightarrow$ Límite*). Evidencia central: el **47.6% de obras en Gobiernos Locales** exhibe alerta de gasto adelantado ($>15\%$), con una brecha mediana de $+11.9\%$ (frente a $+2.0\%$ en el Gobierno Nacional).
7. **Transformaciones y Matriz Analítica:**
   * Creación de variables de negocio: `pct_ejecucion_financiera`, `brecha_gasto_avance`, `flag_gasto_adelantado`, `tiempo_maduracion_anios`, `gasto_anual_promedio`.
   * Discretización ordinal (`pd.cut`) de escala de inversión.
   * Codificación One-Hot de `nivel_gobierno` (`drop_first=True`).
   * Escalamiento Min-Max $[0, 1]$ y estandarización Z-Score ($\mu=0, \sigma=1$) aplicado **exclusivamente sobre variables cuantitativas continuas**.
   * Matriz Analítica final de **1,978 filas × 13 columnas** lista para minería de datos, con el identificador aislado para trazabilidad.
8. **Plan de Modelado (Hito 3):** Hoja de ruta para clustering (K-Means, DBSCAN), detección de anomalías (Isolation Forest) y validación externa cruzada con el portal INFOBRAS de la Contraloría.

---

## 5. Instrucciones de Reproducibilidad

### Opción A — Ejecución del Notebook en Jupyter / Google Colab:
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

### Opción B — Regeneración de Documento Oficial:
Para regenerar el informe oficial en formato Word con tablas y cajas metodológicas:
```bash
python Entrega_2/generar_documento_word_hito2.py
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
