# Modelo latente multimodal de envejecimiento celular en fibroblastos

![GitHub last commit](https://img.shields.io/badge/status-en%20desarrollo-yellow)
![MVP-1](https://img.shields.io/badge/MVP--1-cerrado-success)
![MVP-2](https://img.shields.io/badge/MVP--2-en%20curso-orange)
![Modalidades](https://img.shields.io/badge/modalidades-imagen%20%7C%20RNA%20%7C%20metilaci%C3%B3n%20%7C%20biomarcadores-blue)
![Repositorio](https://img.shields.io/badge/fusi%C3%B3n-concat%20%2B%20mask-informational)

![Diagrama del proyecto](figures/diagrama_modelo_latente.png)

Repositorio del proyecto de tesis orientado a la **inferencia del envejecimiento celular** mediante un **modelo latente multimodal**. El objetivo es aprender una representación latente `z` que integre información de **imágenes brightfield**, **RNA-seq**, **metilación** y **biomarcadores escalares** para modelar el estado funcional de fibroblastos humanos primarios y servir como base para la predicción de **hallmarks of aging** y un **Índice Continuo de Daño (ICD)**.

---

## Tabla de contenido

- [1. Resumen del proyecto](#1-resumen-del-proyecto)
- [2. Estado actual](#2-estado-actual)
- [3. Arquitectura del modelo](#3-arquitectura-del-modelo)
- [4. Estructura del repositorio](#4-estructura-del-repositorio)
- [5. Datasets y fuentes](#5-datasets-y-fuentes)
- [6. Notebooks](#6-notebooks)
- [7. Resultados y carpeta `results/`](#7-resultados-y-carpeta-results)
- [8. Instalación](#8-instalación)
- [9. Uso](#9-uso)
- [10. Convenciones del proyecto](#10-convenciones-del-proyecto)
- [11. Estado técnico real y limitaciones](#11-estado-técnico-real-y-limitaciones)
- [12. Referencias](#12-referencias)

---

## 1. Resumen del proyecto

El proyecto está construido por etapas. Primero se entrenan **encoders unimodales** por modalidad. Después se alinean y se fusionan sus embeddings para obtener un espacio latente común `z_fused`, robusto a modalidades faltantes.

**Objetivo general**

Construir un modelo latente multimodal `z` que permita:

- predecir **edad replicativa** (`PDL`) desde imágenes brightfield,
- entrenar encoders ómicos que produzcan embeddings útiles para fusión,
- integrar modalidades con datos faltantes reales,
- aproximar señales alineadas con **hallmarks of aging**,
- sentar la base para un **Índice Continuo de Daño (ICD 0–1)**.

**Célula modelo**

- Fibroblasto humano primario.

**Dataset principal**

- *Cellular Lifespan Study* de Sturm et al. (Scientific Data, 2022).

---

## 2. Estado actual

### MVP-1 — Encoder visual

**Estado:** cerrado.

El encoder visual fue entrenado sobre imágenes brightfield 10x para producir:

- un embedding `z_img` de 256 dimensiones,
- una predicción de `PDL`.

**Modelo final:** `A2`

**Resumen de resultados**

- Spearman promedio: **0.733**
- Worst fold: **0.619**
- Mejora contra baseline trivial: **+44.1%**
- Evidencia de *fusion-readiness*: correlación parcial significativa con **mtDNA copy number**

Este encoder queda congelado como componente visual para la etapa multimodal posterior.

### MVP-2 — Encoders ómicos

**Estado:** en curso.

Ya se completaron los baselines clásicos que fijan el techo realista de esta fase.

**Resultados baseline**

- **RNA-seq:** Elastic Net con `Spearman = 0.901`
- **Metilación:** PCA + Elastic Net con `Spearman = 0.772`

**Lectura técnica**

- RNA tiene una señal fuerte y bastante lineal.
- Metilación necesita reducción dimensional previa.
- El valor de los encoders profundos no está en ganar unas centésimas de métrica, sino en generar un **embedding fusionable**.

### MVP-3 — Fusión multimodal

**Estado:** pendiente.

La estrategia actual de fusión es:

- **concat + mask**
- entrenamiento con **drop-modality training**

No se toma PoE como estrategia principal porque el número de muestras con triple intersección fuerte entre modalidades es demasiado bajo para calibrarlo bien.

---

## 3. Arquitectura del modelo

```text
Imágenes brightfield  → Encoder visual (A2)   → z_img
RNA-seq bulk          → Encoder RNA           → z_rna
Metilación EPIC       → Encoder Met           → z_met
Telómero / mtDNA      → Encoder Bio           → z_bio
                                  ↓
                       Fusión concat + mask
                                  ↓
                              z fusionado
                                  ↓
                 Hallmarks of Aging + ICD + PDL_hat
```

**Salidas objetivo del sistema**

- `PDL_hat`: predicción de edad replicativa
- `z_img`, `z_rna`, `z_met`, `z_bio`: embeddings por modalidad
- `z_fused`: embedding multimodal integrado
- scores posteriores para hallmarks e ICD

---

## 4. Estructura del repositorio

```text
project/
├── README.md
├── figures/
│   └── diagrama_modelo_latente.png
├── data/
│   ├── raw/
│   ├── processed/
│   ├── manifests/
│   └── external/
├── notebooks/
│   ├── 01_manifests/
│   ├── 02_mvp1_visual/
│   ├── 03_mvp2_omics/
│   ├── 04_fusion/
│   └── 05_evaluation/
├── src/
│   ├── data/
│   ├── models/
│   ├── training/
│   ├── evaluation/
│   └── utils/
├── results/
│   ├── mvp1/
│   ├── mvp2/
│   ├── fusion/
│   ├── reports/
│   └── figures/
└── docs/
```

### Descripción rápida por carpeta

#### `figures/`
Contiene figuras del proyecto para documentación, reportes y README.

#### `data/raw/`
Datos descargados o copiados en estado fuente.

Ejemplos:
- CSV centrales del estudio
- matrices RNA-seq
- matrices de metilación
- metadata GEO
- archivos fuente de biomarcadores

#### `data/processed/`
Datos filtrados, normalizados o transformados para análisis y modelado.

#### `data/manifests/`
Carpeta crítica del pipeline. Aquí viven los **manifests** que actúan como contrato de datos entre notebooks y modelos.

Regla operativa del proyecto:

> Los modelos consumen manifests, no carpetas sueltas.

#### `data/external/`
Datasets complementarios para validación externa o generalización.

#### `notebooks/`
Trabajo exploratorio, ingeniería de datos, entrenamiento y evaluación por fase.

#### `src/`
Código reusable del proyecto.

#### `results/`
Modelos exportados, reportes, métricas, figuras y embeddings.

#### `docs/`
Documentación adicional, notas metodológicas y material de tesis si se decide centralizarlo aquí.

---

## 5. Datasets y fuentes

### Dataset principal

- **Cellular Lifespan Study** (*Sturm et al., 2022*)

### Modalidades principales del proyecto

- Imágenes brightfield
- RNA-seq bulk
- Metilación EPIC
- Biomarcadores escalares como telómero y mtDNA

### Recursos usados o previstos para validación

- GSE115301
- GSE113957
- GSE130973
- otros recursos auxiliares según la fase de validación externa

### Nota importante

El proyecto tiene *missingness* real entre modalidades. No todas las muestras tienen imagen, RNA y metilación al mismo tiempo. Esa limitación no es un detalle molesto. Es una restricción estructural del problema y condiciona toda la estrategia de fusión.

---

## 6. Notebooks

La carpeta `notebooks/` concentra el trabajo experimental. Una organización razonable es la siguiente.

### `notebooks/01_manifests/`
Construcción y auditoría de manifests.

Tareas típicas:
- parsing de nombres de archivo,
- joins con metadata,
- validación de llaves,
- asignación de folds,
- exportación versionada.

### `notebooks/02_mvp1_visual/`
Notebooks del encoder visual.

Contenido esperado:
- baseline visual,
- ablations,
- ranking loss,
- consistency loss,
- variantes con control de batch,
- modelo final A2,
- exportación de `z_img`.

### `notebooks/03_mvp2_omics/`
Notebooks de RNA, metilación y biomarcadores.

Contenido esperado:
- exploración de matrices,
- selección de genes y CpGs,
- baselines clásicos,
- entrenamiento de encoder RNA,
- entrenamiento de encoder Met,
- batch probes,
- chequeos biológicos.

### `notebooks/04_fusion/`
Integración de embeddings unimodales.

Contenido esperado:
- concat + mask,
- simulación de modalidades faltantes,
- entrenamiento de capa de fusión,
- predicción multimodal de `PDL`, hallmarks e ICD.

### `notebooks/05_evaluation/`
Evaluación final, generalización y reportes.

Ejemplos:
- métricas por fold,
- análisis de error,
- visualizaciones del espacio latente,
- pruebas de generalización externa,
- interpretabilidad.

---

## 7. Resultados y carpeta `results/`

La carpeta `results/` debe concentrar artefactos reproducibles del proyecto.

### Contenido esperado

- pesos de modelos entrenados,
- archivos `.pt`, `.pth` o equivalentes,
- embeddings exportados,
- reportes de métricas,
- figuras,
- tablas de evaluación,
- salidas intermedias de validación.

### Ejemplo de organización

```text
results/
├── mvp1/
│   ├── final/
│   ├── baseline/
│   ├── ablations/
│   └── figures/
├── mvp2/
│   ├── baselines/
│   ├── rna/
│   ├── met/
│   └── figures/
├── fusion/
├── reports/
└── figures/
```

### Estado actual esperado

- `mvp1/` debe contener el modelo A2 final y sus artefactos.
- `mvp2/` debe contener baselines, pruebas de encoder RNA y pruebas posteriores de metilación.

---

## 8. Instalación

### Requisitos

- Python 3.13
- entorno virtual recomendado
- dependencias científicas estándar para manejo de datos, modelado y entrenamiento

### Crear entorno virtual

```bash
python3.13 -m venv .venv
source .venv/bin/activate
```

### Instalar dependencias

Si ya existe un archivo `requirements.txt`:

```bash
pip install -r requirements.txt
```

Si el repositorio todavía no tiene un `requirements.txt` consolidado, esa es una deuda técnica razonable a resolver antes de publicar el repo como proyecto reproducible.

---

## 9. Uso

La secuencia lógica de uso del repositorio es esta.

### 1. Construir manifests

```bash
jupyter lab
```

Abrir los notebooks de `01_manifests/` y generar los manifests versionados a partir de los datos fuente.

### 2. Ejecutar MVP-1 o cargar artefactos ya entrenados

- entrenar o reutilizar el encoder visual A2,
- exportar `z_img`.

### 3. Ejecutar MVP-2

- correr baselines de RNA y metilación,
- entrenar encoders ómicos,
- exportar `z_rna` y `z_met`.

### 4. Entrenar fusión

- integrar embeddings disponibles,
- usar máscara de modalidades,
- entrenar con *drop-modality*.

### 5. Evaluar

- medir métricas por fold,
- revisar batch leakage,
- inspeccionar correlaciones biológicas,
- preparar reportes.

### Ejemplo de flujo mínimo esperado

```text
raw data → manifests → encoder visual / encoder ómico → embeddings → fusión → evaluación
```

---

## 10. Convenciones del proyecto

### 1. Manifests primero
Los modelos deben leer manifests. No deben depender de rutas ad hoc escritas a mano dentro de cada notebook.

### 2. Splits con anti-leakage
Los splits deben respetar estructura biológica y experimental. Un split aleatorio cómodo puede dar métricas bonitas y ciencia basura. Mala combinación.

### 3. Versionado de artefactos
Los exports relevantes deben llevar timestamp o convención clara de versión.

### 4. Métricas no triviales
No basta con reportar una sola métrica. El proyecto necesita:
- desempeño predictivo,
- estabilidad entre folds,
- chequeos de batch,
- lectura biológica mínima.

---

## 11. Estado técnico real y limitaciones

Este repositorio no está en una fase de producto terminado. Está en una fase de investigación activa con resultados ya defendibles en MVP-1 y señal sólida para MVP-2.

### Limitaciones relevantes

- Existe contaminación residual por protocolo o dominio en la parte visual.
- La intersección fuerte entre modalidades es baja.
- `PDL` no es equivalente a edad cronológica.
- Metilación requiere reducción dimensional agresiva para ser tratable.
- Los encoders profundos en ómicas deben justificarse por valor de representación, no por fe religiosa en redes neuronales.

### Decisión metodológica importante

La fusión principal actual es **concat + mask**. No es la opción más elegante sobre papel. Es la opción más honesta para el régimen de datos real.

---

## 12. Referencias

### Artículo base del dataset principal

- Sturm et al. (2022). *Cellular Lifespan Study*. Scientific Data.

### Conceptos centrales del proyecto

- Hallmarks of Aging
- representación latente multimodal
- predicción de PDL en fibroblastos
- integración multimodal con modalidades faltantes

---

## Nota final

Este repositorio documenta un proyecto de tesis en desarrollo. El foco no es solo obtener métricas altas. El foco es construir un pipeline defendible, reproducible y útil para una inferencia multimodal seria del envejecimiento celular.
