# Data Hunters — Seguimiento Presupuestal y Avance Físico de Proyectos de Inversión Pública (Perú)

Proyecto del curso **Data Mining** (Universidad del Pacífico, docente Soledad Espezúa Llerena) desarrollado por el equipo **Data Hunters**: Fernando Torres, Romy Tipacti y Arturo Alvarez.

## Problema

El **MEF** (Ministerio de Economía y Finanzas del Perú) registra cuánto presupuesto tiene asignado cada proyecto de inversión pública y cuánto se ha devengado (ejecución financiera a través del SIAF), pero esa cifra por sí sola no dice si la obra realmente está avanzando en el terreno. Por otro lado, el historial de procesos de selección del **OSCE / SEACE** registra el avance físico y contractual mes a mes según las valorizaciones técnicas de los supervisores de obra, pero como una base de datos completamente separada, sin relación explícita con lo financiero.

**Objetivo del Proyecto:** integrar ambas fuentes para cuantificar la **Brecha Gasto - Avance** (*"¿este proyecto ya gastó el 80% o 100% del presupuesto pero solo tiene 20% de avance físico?"*), permitiendo identificar de forma temprana proyectos de inversión pública en riesgo de paralización, sobreejecución presupuestal o abandono de obra.

**¿A quién le sirve?** A quienes supervisan las inversiones públicas: equipos de seguimiento del MEF, Órganos de Control Institucional (OCI), Unidades Ejecutoras y auditores de la **Contraloría General de la República del Perú**.

## Fuentes de datos

Ambas fuentes provienen del Portal de Datos Abiertos del MEF:

| Fuente | Dataset | Contenido | Enlace |
|---|---|---|---|
| **Seguimiento_PI.csv** | Seguimiento de Proyectos de Inversión | Asignación presupuestal y ejecución financiera (PIM, devengado, gasto acumulado) por año fiscal, pliego y fuente de financiamiento. Proviene del SIAF y Banco de Inversiones (Invierte.pe). | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/2026-Seguimiento-PI.csv) |
| **Proceso_Selección.csv** (OSCE) | Proceso de Selección de Inversiones | Historial mensual de convocatorias, licitaciones, etapas contractuales y avance físico porcentual acumulado de obra. Se nutre de la interoperabilidad MEF–SEACE/OSCE. | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/Proceso_Selecccion_Diccionario.csv) |

**Llave de integración:** `codigo_proyecto`, resultado de homologar `PRODUCTO_PROYECTO` (Seguimiento_PI) y `CODIGO_UNICO` (Proceso_Selección).

**Unidad de análisis:** Cada fila representa un **Proyecto de Inversión Pública (obra o intervención específica)**, identificado por su Código Único de Inversión (**CUI**).

---

## Estructura del Repositorio

El proyecto está organizado de manera modular:

```
├── Clases/                                   # Material teórico y laboratorios de clase (Dra. Soledad Espezúa)
│   ├── (DM)Integracion_Unidad_Analisis_Clinica_sol.ipynb
│   ├── (DM)Faltantes_Codificacion.ipynb
│   ├── (DM)Continuacion_Transformacion.ipynb
│   └── [4]Sesión5(DM).pdf, [5]Sesión8(DM).pdf, [6]Sesión9(DM).pdf, [6]Sesión10(DM).pdf
│
├── Semana 1/
│   ├── Integracion_Sem_1.ipynb              # Exploración inicial e integración preliminar
│   └── diccionario_de_datos_entregable.xlsx
│
├── Semana 2/
│   └── (DM)limpieza_datos._Data_Hunters.ipynb  # Diagnóstico preliminar de calidad y limpieza documentada
│
├── Semana 3 - Entrega 1/                     # Primera entrega formal (Hito Formativo 1)
│   ├── Código_Fuente_Documentado/
│   │   ├── Entrega 1 - Data Hunters.ipynb
│   │   ├── diccionario_de_datos.xlsx
│   │   └── diccionario_nombres_claros.csv
│   ├── Entrega_1_Hito_Formativo_Data_Hunters.docx
│   └── Informe_Fuentes_de_Datos.pdf
│
└── Entrega_2/                                # SEGUNDA ENTREGA FORMAL (HITO 2: 40% DEL PROYECTO)
    ├── Entrega_2_Hito_Formativo_Data_Hunters.docx # Documento oficial formal (Word con formato académico UP)
    ├── Entrega_2_Data_Hunters.ipynb         # Notebook documentado y reproducible (Estructura Diapositiva 4)
    ├── base_integrada_proyectos.csv         # Base integrada a nivel de Proyecto CUI (52,480 filas × 19 col)
    ├── matriz_analitica.csv                 # Primera Matriz Analítica para Minería (1,978 filas × 17 col)
    ├── matriz_analitica_minmax.csv          # Matriz Analítica normalizada Min-Max [0, 1]
    ├── matriz_analitica_zscore.csv          # Matriz Analítica estandarizada Z-Score (media 0, std 1)
    ├── diccionario_matriz_analitica.csv     # Diccionario de variables de la matriz analítica
    ├── graficos/                            # Visualizaciones del EDA en alta resolución (300 DPI)
    │   ├── g1_avance_por_nivel_gobierno.png
    │   ├── g2_dispersion_gasto_vs_avance.png
    │   ├── g3_distribucion_brecha.png
    │   └── g4_proyectos_adelantados_por_gobierno.png
    ├── generar_datos_hito2.py               # Script automatizado de limpieza, agregación y escalamiento
    └── generar_documento_word_hito2.py      # Script generador del reporte formal en Word
```

---

## Estructura del Código Fuente Documentado (Hito 2)

El notebook [`Entrega_2/Entrega_2_Data_Hunters.ipynb`](Entrega_2/Entrega_2_Data_Hunters.ipynb) sigue al pie de la letra los 6 apartados solicitados en la **diapositiva 4** de la pauta del Hito 2:

1. **Carga e inspección de las fuentes:**
   - Detección flexible de rutas de datos (funciona en entorno local, Google Colab o aula de clases).
   - Inspección de dimensiones, tipos de datos y primeras filas.
2. **Validación de claves, duplicados y granularidad:**
   - Eliminación de **871,679 duplicados exactos** en SEACE.
   - Diagnóstico y exclusión de **197 códigos presupuestales genéricos** (como `2001621` *"Estudios de Pre-Inversión"*, `2000634` *"Fortalecimiento"*, `2005230` *"Centros Educativos"*), compartidos por más de 5 entidades independientes y que acumulaban 479,015 filas sin CUI de obra física en SEACE.
   - Agregación con `groupby("PRODUCTO_PROYECTO")` para reducir `Seguimiento_PI` a **52,480 proyectos reales individualizados** (**1 fila = 1 CUI**).
3. **Diagnóstico y tratamiento de faltantes y valores atípicos:**
   - Normalización de etapas de contratación a mayúsculas homogéneas.
   - Depuración de avances fuera de rango $[0, 100]\%$, costos $\le 0$ y periodos futuros a `NaN`/`NaT`.
   - Reducción de `Proceso_Selección` a **204,162 proyectos únicos** con su último avance válido.
4. **Integración y comprobación de registros emparejados/no emparejados (Slide 5):**
   - Evidencia de granularidad: se pasa de partidas anuales y reportes mensuales a 1 fila por CUI.
   - Validación estricta `1 a 1` (`validate='one_to_one'`) mediante `left join`.
   - **Auditoría de cruce:** **35,472 proyectos emparejados (`both`, 67.59%)** y 17,008 proyectos en `left_only` (obras por Administración Directa, compras $<8$ UIT o formulación previa sin SEACE).
5. **EDA y visualizaciones relevantes:**
   - 4 gráficos clave interpretados bajo la estructura pedagógica de la Universidad del Pacífico:
     **Observación → Evidencia → Interpretación → Límite**.
   - Hallazgo central: el **47.6% de proyectos de Gobiernos Locales** presenta alerta de gasto adelantado ($>15\%$), con una brecha mediana de $+11.9\%$ (frente a $+2.0\%$ del Gobierno Nacional).
6. **Transformaciones aplicadas y construcción de la matriz analítica:**
   - Variables derivadas con sentido de negocio: `pct_ejecucion_financiera`, `brecha_gasto_avance` (target de desfase), `tiempo_maduracion_anios`, `gasto_anual_promedio`, `flag_gasto_adelantado`.
   - Discretización (`pd.cut`): `tamano_inversion` y `nivel_desfase`.
   - Detección de outliers (IQR y Z-Score) justificando la conservación de megaproyectos y casos extremos.
   - Codificación One-Hot de `nivel_gobierno` y escalamiento Min-Max $[0, 1]$ y Z-Score ($\mu=0, \sigma=1$).
   - Matriz Analítica final de **1,978 proyectos CUI** en ejecución activa, conservando `codigo_proyecto` solo como índice de trazabilidad.

---

## Cómo reproducir el Hito 2 en el Aula de Clases

Para que la docente pueda replicar el código de inmediato en el salón de clases:

### Opción A — Ejecución del Notebook en Jupyter / Google Colab:
1. Clonar el repositorio:
   ```bash
   git clone https://github.com/RomyUP1411/data-mining-project.git
   ```
2. Si los archivos CSV crudos (`Seguimiento_PI.csv` y `Proceso_Seleccion.csv`) se colocan en la misma carpeta del notebook o en `data/`, el notebook los procesará automáticamente.
3. Si no se descargan los archivos crudos (que pesan 1.5 GB), el notebook contiene salidas pre-renderizadas completas con todos los gráficos, tablas y estadísticas para su inspección inmediata.
4. Abrir y ejecutar:
   ```bash
   jupyter notebook Entrega_2/Entrega_2_Data_Hunters.ipynb
   ```

### Opción B — Ejecución de los Scripts en Terminal:
1. Regenerar los datasets y gráficos:
   ```bash
   python Entrega_2/generar_datos_hito2.py
   ```
2. Regenerar el informe oficial en Word:
   ```bash
   python Entrega_2/generar_documento_word_hito2.py
   ```

---

## Estado del Proyecto

- [x] **Hito 1:** Planteamiento del problema, exploración inicial y viabilidad.
- [x] **Hito 2:** Preparación de datos, corrección de granularidad (agregación 1:1 por CUI), exclusión de códigos genéricos, EDA formal y primera matriz analítica.
- [ ] **Hito 3:** Modelado de Minería de Datos (Clustering K-Means / DBSCAN y Detección de Anomalías con Isolation Forest).

## Equipo

**Data Hunters** — Fernando Torres, Romy Tipacti, Arturo Alvarez  
Curso: Data Mining — Universidad del Pacífico  
Docente: Soledad Espezúa Llerena (s.espezua@up.edu.pe)
