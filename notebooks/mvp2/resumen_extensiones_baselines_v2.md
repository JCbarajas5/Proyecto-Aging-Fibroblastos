# Notebook Baselines v2 Extended — Resumen de Extensiones

**Archivo:** `mvp2_02_baselines_clasicos_v2_extended.ipynb`  
**Versión:** 2.0 (extended)  
**Fecha:** 29 de marzo de 2026

---

## Secciones originales (1-17) — SIN MODIFICAR

Las primeras 17 secciones permanecen **idénticas** al notebook v1:

1. Configuración y dependencias
2. Rutas y configuración
3. Carga manifest MVP-2
4. Carga matrices ómicas
5. Join manifest ↔ matrices
6. Preparar datasets por fold
7. **Experimento 1a:** Elastic Net RNA → PDL
8. **Experimento 1b:** XGBoost RNA → PDL
9. **Experimento 1c:** Elastic Net Met crudo → PDL
10. **Experimento 1d:** PCA + Elastic Net Met → PDL
11. Comparación RNA: EN vs XGB
12. Comparación Met: Crudo vs PCA
13. Decisión XGBoost Met (opcional)
14. Visualizaciones (scatter plots)
15. PCA varianza explicada
16. Guardar resultados
17. Metadata y versionado

---

## NUEVAS SECCIONES DIAGNÓSTICAS (18-21)

### **Sección 18: Diagnóstico composición por fold**

**Objetivo:** Documentar cómo se distribuyen variables clave entre folds.

**Análisis incluidos:**
1. **Cell_lines por fold** (RNA)
   - Tabla con conteo de muestras por cell_line en cada fold
   - Identifica si SURF1 mutation está concentrado en un fold

2. **Distribución PDL por fold** (RNA y Met)
   - Estadísticas descriptivas (mean, std, min, max, percentiles)
   - Detecta si folds tienen PDL promedio distinto

3. **Study_part por fold** (RNA y Met)
   - Conteo por study_part en cada fold
   - Identifica desbalance de protocolo experimental

4. **Clinical_condition por fold** (Met)
   - Distribución Normal vs SURF1_Mutation
   - Verifica si Fold 0 es 100% Normal

**Output:**
- Tablas diagnósticas impresas
- Interpretación de limitaciones estructurales

**Valor:**
- Evidencia para interpretar worst fold
- Documenta trade-off GroupKFold (anti-leakage vs balance)
- Fundamenta batch probe obligatorio en Experimento 2

---

### **Sección 19: Top genes Elastic Net RNA**

**Objetivo:** Identificar genes más importantes según modelo baseline.

**Análisis incluidos:**
1. **Entrenar Elastic Net en TODOS los datos RNA**
   - No por fold (maximiza datos)
   - Extrae coeficientes de modelo final

2. **Top 50 genes por coeficiente absoluto**
   - Tabla con gene, coef, abs_coef
   - Ordenados por importancia

3. **Overlap con hallmark genes**
   - Verifica cuántos de los 40 hallmark genes aparecen en top 50
   - Interpreta si señal es biológicamente coherente

**Output:**
- `top_genes_elasticnet_rna.csv` (top 50 genes)
- Tabla impresa con genes + coeficientes
- Verificación overlap hallmark

**Valor:**
- **Interpretabilidad biológica** para tesis
- Valida que genes seleccionados son relevantes para aging
- Puede revelar genes novel no anticipados

---

### **Sección 20: Nota Spearman vs R²**

**Objetivo:** Explicar por qué R² negativos NO invalidan el baseline.

**Contenido:**
1. **Definición métricas**
   - Spearman: correlación de ranking (métrica principal)
   - R²: proporción varianza explicada (calibración secundaria)

2. **Ejemplo Met crudo Fold 0**
   - Spearman 0.921 (excelente ranking)
   - R² -5.197 (calibración catastrófica)
   - Interpretación: ordena bien, pero magnitudes desviadas

3. **Tabla comparativa: PCA mejora ambas**
   - Antes/después PCA para los 3 folds
   - Demuestra que PCA no solo comprime, también calibra

4. **Defensa para tesis**
   - 6 puntos argumentativos
   - Argumento clave: "Objetivo es inferir estado de aging (ranking), no predecir PDL absoluto"

**Valor:**
- **Anticipar crítica de reviewers** sobre R² negativos
- Justificar que Spearman es métrica principal
- Documentar que PCA mejora calibración

---

### **Sección 21: Tabla resumen decisiones**

**Objetivo:** Consolidar decisiones metodológicas para MVP-2.

**Contenido:**
1. **Tabla comparativa RNA vs Met**
   - Baseline elegido
   - Señal detectada (fuerte/moderada, lineal/PCA)
   - Worst fold Spearman
   - Decisión encoder (avanzar/precaución)
   - Target Spearman (90% baseline)

2. **Riesgos estructurales identificados (4 items)**
   - Distribución PDL no homogénea
   - Study_part desbalanceado
   - R² negativos en Met crudo
   - Folds desbalanceados (Fold 2: 47 muestras)
   - **Cada riesgo con consecuencia y mitigación**

3. **Justificación encoders profundos**
   - ¿Por qué Elastic Net no sirve para fusión?
   - ¿Qué aporta el MLP?
   - Criterio de éxito (4 puntos)
   - **Argumento clave:** Valor está en z fusionable, no en superar EN

4. **Conclusión final**
   - Baselines cumplen función control de realidad
   - Upper bounds establecidos
   - AVANZAR a Experimento 2 con expectativas realistas

**Output:**
- `decisiones_avance_mvp2.csv` (tabla comparativa)
- `riesgos_y_justificacion.txt` (texto consolidado)

**Valor:**
- **Decisión formal documentada**
- Transparencia metodológica
- Argumentación defensiva para tesis

---

## Archivos generados (adicionales)

El notebook v2 extended genera **3 archivos adicionales** en `results/mvp2_baselines/`:

1. `top_genes_elasticnet_rna.csv`
   - Top 50 genes más importantes
   - Columnas: gene, coef, abs_coef

2. `decisiones_avance_mvp2.csv`
   - Tabla comparativa RNA vs Met
   - Decisión formal de avance

3. `riesgos_y_justificacion.txt`
   - Riesgos estructurales (texto plano)
   - Justificación encoders profundos

**Total archivos baseline (v2):**
- 4 JSON métricas (EN RNA, XGB RNA, EN Met crudo, EN Met PCA)
- 4-5 CSV predicciones (por modelo)
- 5 PNG plots (scatter × 4, PCA variance)
- 1 CSV top genes
- 1 CSV decisiones
- 1 TXT riesgos
- 1 JSON metadata

---

## Diferencias clave v1 vs v2

| Aspecto | v1 (original) | v2 (extended) |
|---------|---------------|---------------|
| **Secciones** | 17 | 21 |
| **Experimentos** | 1a-1e (sin cambios) | 1a-1e (idéntico) |
| **Diagnóstico folds** | ❌ No | ✅ Sí (Sección 18) |
| **Top genes** | ❌ No | ✅ Sí (Sección 19) |
| **Nota Spearman/R²** | ❌ No | ✅ Sí (Sección 20) |
| **Tabla decisiones** | ❌ No | ✅ Sí (Sección 21) |
| **Archivos output** | ~10 | ~13 |
| **Valor científico** | Métricas | Métricas + interpretabilidad |

---

## Uso recomendado

**v1 (original):** Control de realidad rápido, métricas core  
**v2 (extended):** Artefacto completo para tesis, con diagnósticos y justificación

**Para ejecutar v2:**
1. Ejecutar secciones 1-17 (idéntico a v1)
2. Ejecutar secciones 18-21 (extensiones diagnósticas)
3. Total tiempo estimado: +15 min vs v1

**Para tesis:**
- Usar v2 como artefacto principal
- Secciones 18-21 van en "Análisis de limitaciones" o "Diagnóstico dataset"
- Top genes van en "Interpretabilidad biológica"

---

## Siguiente paso

Con baselines v2 completados → **Generar Experimento 2: Encoder RNA baseline**

**Notebook siguiente:** `mvp2_03_encoder_rna_baseline.ipynb`

Con:
- MLP simple (2 capas), dropout 0.5/0.3
- Batch probe ΔAUC(study_part) bruto + condicional
- UMAP, correlaciones cross-modal
- Target: Spearman ≥ 0.811, worst fold ≥ 0.76

---

**Fin del resumen.**
