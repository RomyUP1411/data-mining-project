# Data Hunters — Seguimiento Presupuestal y Avance Físico de Proyectos de Inversión Pública (Perú)

Proyecto del curso **Data Mining** (Universidad del Pacífico, docente Soledad Espezúa) desarrollado por el equipo **Data Hunters**: Fernando Torres, Romy Tipacti y Arturo Alvarez.

## Problema

El **MEF** (Ministerio de Economía y Finanzas del Perú) sabe cuánto presupuesto tiene cada proyecto de inversión pública y cuánto se ha gastado (ejecución financiera), pero esa cifra por sí sola no dice si la obra realmente está avanzando. Por otro lado, el historial de procesos de selección sí registra el avance físico y contractual mes a mes, pero como una tabla completamente separada, sin relación explícita con lo financiero.

**Objetivo:** integrar ambas fuentes para poder responder preguntas como *"¿este proyecto ya gastó el 80% del presupuesto pero solo tiene 20% de avance físico?"*, y así identificar proyectos de inversión pública cuyo patrón de ejecución presupuestal se aleja del comportamiento típico (subejecución, sobreejecución inusual o posible paralización).

**¿A quién le sirve?** A quien supervisa estos proyectos: un equipo de seguimiento de inversiones, una contraloría, o el propio equipo del proyecto, que necesita detectar a tiempo los casos donde se gasta presupuesto sin que la obra avance al mismo ritmo.

## Fuentes de datos

Ambas fuentes provienen del Portal de Datos Abiertos del MEF:

| Fuente | Dataset | Contenido | Enlace |
|---|---|---|---|
| **Seguimiento_PI.csv** | Seguimiento de Proyectos de Inversión | Avance financiero y estado situacional de cada proyecto (presupuesto, monto ejecutado) por año, pliego y fuente de financiamiento. Proviene del Sistema de Seguimiento de Inversiones (SSI) y el Banco de Inversiones (Invierte.pe). | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/2026-Seguimiento-PI.csv) |
| **Proceso_Selección.csv** (OSCE) | Proceso de Selección de Inversiones | Historial mensual de convocatorias, licitaciones y avance físico/contractual de cada obra. Se nutre de la interoperabilidad MEF–SEACE/OSCE. | [datosabiertos.mef.gob.pe](https://fs.datosabiertos.mef.gob.pe/datastorefiles/Proceso_Selecccion_Diccionario.csv) |

**Llave de integración:** `PRODUCTO_PROYECTO` (Seguimiento_PI) ↔ `CODIGO_UNICO` (Proceso_Selección) — renombrada a `codigo_proyecto` en ambas tablas para facilitar el cruce.

**Unidad de análisis:** cada fila representa un **proyecto de inversión pública** (obra o intervención específica), identificado por su Código Único de Inversión (CUI).

## Estructura del repositorio

El proyecto está organizado por semanas de avance del curso, cada una construyendo sobre la anterior:

```
├── Semana 1/
│   ├── Integracion_Sem_1.ipynb              # Primera exploración e integración (laboratorio guiado)
│   ├── diccionario_de_datos_entregable.xlsx # Diccionario de datos inicial
│   └── requirements.txt
│
├── Semana 2/
│   └── (DM)limpieza_datos._Data_Hunters.ipynb  # Limpieza de datos e integración documentada
│
└── Semana 3 - Entrega 1/                     # Primera entrega formal (Hito Formativo)
    ├── Código_Fuente_Documentado/
    │   ├── Entrega 1 - Data Hunters.ipynb        # Notebook final: EDA + integración + gráficos + hallazgos
    │   ├── diccionario_de_datos.xlsx              # Diccionario de datos (49 columnas, ambas fuentes)
    │   ├── diccionario_nombres_claros.csv         # Mapeo nombre original → nombre claro por variable
    │   └── requirements.txt
    ├── Entrega_1_Hito_Formativo_Data_Hunters.docx # Informe de la entrega
    ├── Informe_Fuentes_de_Datos.docx / .pdf       # Ficha descriptiva de las fuentes de datos
```

> **Nota:** los archivos CSV crudos (`Seguimiento_PI.csv`, `PROCESO_SELECCION.csv`, ~1 GB y ~5.7 millones de filas) y las bases integradas generadas no se versionan en este repositorio por su tamaño; los notebooks asumen que se descargan localmente desde los enlaces de la sección anterior antes de ejecutarse.

## Qué hace cada notebook

### Semana 1 — Integración inicial (laboratorio guiado)
Primer contacto con ambas fuentes: inspección de tamaño y tipos de datos, `describe()`, revisión de valores faltantes y de valores distintos por variable, identificación de la llave de integración (`PRODUCTO_PROYECTO` ↔ `CODIGO_UNICO`), un primer `merge` (left join) y un análisis de calidad de datos que documenta los problemas encontrados: datos faltantes, datos duplicados, formatos no uniformes/valores inválidos y datos redundantes.

### Semana 2 — Limpieza de datos documentada
Profundiza el diagnóstico de calidad y aplica las correcciones:
- **Faltantes disfrazados:** `SECTOR`/`PLIEGO` traían un espacio en blanco en vez de venir vacíos (faltante estructural de los gobiernos locales, que no tienen sector nacional asignado) — se convierten a `NaN` explícito.
- **Duplicados:** se identifican y eliminan >100 mil filas exactamente duplicadas en `Proceso_Selección`, distinguiéndolas de la repetición legítima del historial mensual por proyecto.
- **Categorías inconsistentes:** `DES_ETAPA` mezclaba mayúsculas/minúsculas, tildes y códigos entre paréntesis para la misma etapa del proyecto — se homologa a una sola convención.
- **Tipos mal detectados:** `VAL_META_CAPAC` venía como texto por usar coma decimal; se convierte a numérico.
- **Valores fuera de rango:** `AVANCE` con valores imposibles (órdenes de 10¹⁴, mezcla de escalas), `COSTO_INVERSION` negativo y `PERIODO` con fechas hasta 15 años en el futuro — se marcan como faltantes explícitos, sin imputarlos.
- **Reducción a una fila por proyecto:** dado que `Proceso_Selección` trae hasta ~7,600 filas de historial por proyecto, se construye un snapshot con el período más reciente por `CODIGO_UNICO` antes de cruzar, evitando una explosión de filas en el merge (de lo contrario, el cruce directo genera ~13 millones de filas duplicadas).
- **Integración final** (`left join` preservando la base MEF íntegra) y exportación de la base integrada (~697 mil filas × 52 columnas) junto con una tabla de decisiones de limpieza justificadas.

### Semana 3 — Entrega 1 (Hito Formativo)
Notebook final y documentado que retoma la limpieza de la Semana 2 y añade:
1. **Diccionario de nombres claros:** renombrado de las 49 columnas originales (`ANO_EJE`, `SEC_EJEC`, etc.) a nombres legibles (`anio_ejecucion`, `seccion_ejecutora_cod`, etc.), preservando `codigo_proyecto` como llave común en ambas tablas.
2. **Definición de la unidad de análisis** y de las variables clave para comparar, agrupar, asociar y detectar patrones atípicos.
3. **Auditoría de la clave de integración** (`codigo_proyecto`): tabla de validación con conteo de valores únicos, repeticiones promedio/máximas y explicación de por qué cada fuente repite la llave por diseño (año/pliego/fuente de financiamiento en un caso, historial mensual en el otro).
4. **Cruce con `how="outer"`** para auditar coincidencias, y luego **integración final con `how="left"`** preservando todas las filas de `Seguimiento_PI`.
5. **Visualizaciones** (Plotly): resultado del cruce (barras), distribución del avance físico (histograma), avance físico por nivel de gobierno (boxplot) y monto ejecutado vs. avance físico (scatter, con color por nivel de gobierno y tamaño por costo de inversión).
6. **Hallazgos principales**, entre ellos:
   - El match entre fuentes es del **24.8% por fila** pero del **67.3% por proyecto** (35,542 de 52,800 proyectos), porque los proyectos sin cruce son, en promedio, los que más se repiten en `Seguimiento_PI`.
   - El **91%** de los registros que sí cruzan tienen `avance_fisico_pct` vacío, porque este campo solo se registra cuando `etapa_proyecto = 'EJECUCION'`.
   - El código `2001621` ("Estudios de Pre-Inversión") aparece **82,864 veces** (11.9% de las filas de `Seguimiento_PI`) y no es un proyecto real, sino un código presupuestal genérico que conviene tratar aparte.
   - El avance físico promedio es similar entre niveles de gobierno (nacional 79.9%, local 79.2%, regional 76.8%) pese a que el gobierno nacional ejecuta ~10x más presupuesto por proyecto que el local — gastar más no garantiza más avance.

## Diccionario de datos

El diccionario de datos completo (49 variables entre ambas fuentes: nombre original, nombre claro, fuente, descripción, tipo, escala de medición y observaciones de calidad) está disponible en:
- [`Semana 3 - Entrega 1/Código_Fuente_Documentado/diccionario_de_datos.xlsx`](Semana%203%20-%20Entrega%201/C%C3%B3digo_Fuente_Documentado/diccionario_de_datos.xlsx)
- [`Semana 3 - Entrega 1/Código_Fuente_Documentado/diccionario_nombres_claros.csv`](Semana%203%20-%20Entrega%201/C%C3%B3digo_Fuente_Documentado/diccionario_nombres_claros.csv)

## Cómo ejecutar

1. Descargar los CSV crudos desde los enlaces de la sección [Fuentes de datos](#fuentes-de-datos) y colocarlos en la misma carpeta que el notebook a ejecutar (`Seguimiento_PI.csv` y `PROCESO_SELECCION.csv`).
2. Instalar las dependencias del notebook correspondiente:

   ```bash
   pip install -r "Semana 3 - Entrega 1/Código_Fuente_Documentado/requirements.txt"
   ```

3. Abrir el notebook con Jupyter (`jupyter notebook`) o en Google Colab y ejecutar las celdas en orden.

### Dependencias principales

| Paquete | Uso |
|---|---|
| `pandas` | Carga, limpieza e integración de datos |
| `numpy` | Manejo de valores faltantes y operaciones numéricas |
| `plotly` | Visualizaciones interactivas (Semana 3) |
| `matplotlib` | Visualizaciones estáticas (Semana 2) |
| `openpyxl` | Lectura/escritura de archivos Excel (diccionario de datos) |
| `nbformat`, `ipykernel` | Soporte de notebooks Jupyter |

## Estado del proyecto

- ✅ Semana 1: exploración e integración inicial
- ✅ Semana 2: limpieza de datos documentada
- ✅ Semana 3: Entrega 1 (Hito Formativo) — integración final, EDA y primeros hallazgos

## Equipo

**Data Hunters** — Fernando Torres, Romy Tipacti, Arturo Alvarez
Curso: Data Mining — Universidad del Pacífico
Docente: Soledad Espezúa (s.espezua@up.edu.pe)
