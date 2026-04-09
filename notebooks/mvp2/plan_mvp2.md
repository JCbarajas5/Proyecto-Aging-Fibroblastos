# Plan MVP-2 Final — Encoders Ómicos
## Versión Ejecutable con Ajustes Finales

**Fecha:** 28 de marzo de 2026  
**Proyecto:** Inferencia del envejecimiento celular mediante modelo latente multimodal  
**Fase:** MVP-2 (Encoders ómicos: RNA, Metilación, Biomarcadores)

---

## Contexto

**MVP-1 (encoder visual) está CERRADO.**  
Modelo A2 final: z_img (256-dim) con Spearman 0.733, congelado para fusión.

**MVP-2 construye encoders ómicos** que:
1. Produzcan embeddings z_rna, z_met, z_bio (cada uno 256-dim)
2. Predigan PDL con Spearman ≥ baseline clásico
3. Se alineen cross-modalmente con z_img
4. Habiliten fusión multimodal (MVP-3)

---

## Ajustes finales incorporados

### A. RankingLoss corregido (crítico)

**❌ Versión peligrosa (repetía error MVP-1):**
```python
Loss = MSE(PDL, PDL_hat) + λ_rank * RankingLoss(z_rna, PDL)
```

**✅ Versión correcta:**
```python
Loss = MSE(PDL, PDL_hat) + λ_rank * RankingLoss(PDL_hat, PDL)
```

**Razón:** Ranking sobre la **predicción escalar**, no sobre la geometría del embedding. Igual que la corrección en MVP-1 A1.

---

### B. Baseline metilación ampliado

**Fase 1 ahora incluye:**
- Elastic Net sobre 10K CpGs crudos
- **PCA → Elastic Net** (separa "señal lineal en PCs" vs "necesitas MLP")
- XGBoost opcional **solo si baselines lineales dejan incertidumbre sobre no-linealidad útil** (no por costo GPU, sino por valor marginal)

---

### C. Batch probe con variable de dominio correcta

**No asumir `study_part` como adversario automático.**  
Variables disponibles en manifest MVP-2: `['study_part']`

Para RNA/Met, el batch probe usará:
- `study_part` (única variable de dominio disponible)
- Si existen en metadata GEO: batch técnico, lote de secuenciación
- Documentar cuál variable es relevante por modalidad

---

### D. Criterios GO/NO-GO como tentativos

Los umbrales (correlación > 0.15, CCA > 0.3) se documentan como:

> **"Criterios operativos tentativos para decidir si fusión vale la pena experimentalmente"**

**No son reglas oficiales de tesis.** Son heurísticas de avance razonables, sujetas a revisión según resultados.

---

### E. Métricas cross-modal robustas

En experimento 5a, **no confiar solo en ρ promedio**.  
Reportar:
- Promedio de correlación por dimensión
- **Mediana** (más robusta a outliers)
- **Top-K dimensiones** con mayor alineación
- **Top componentes canónicas** (CCA)
- Opcional: métrica de **retrieval** (dado z_rna, recuperar z_img correcto en top-5 vecinos)

Esto evita que un promedio bonito oculte que solo 20/256 dimensiones están alineadas.

---

### F. Criterio correlación clocks más estricto

En experimento 3 (encoder Met), el criterio `ρ(clocks) > 0` es **demasiado débil**.

**Criterio ajustado:**
- Correlación en **dirección biológicamente coherente** (z_met vs clocks epigenéticos)
- **Magnitud no trivial** (ρ > 0.3 como tentativo)
- **Consistente en folds** (no solo significativo en 1 fold)
- Idealmente: correlación **parcial** controlando PDL

---

### G. Regla de disciplina experimental

**No abrir experimentos 6a (VAE) ni 6b (Sparse) hasta que 2 (RNA MLP) y 3 (Met MLP) tengan:**
- Reporte completo de métricas
- Plots (scatter, UMAP, batch probe)
- Diagnóstico de overfitting
- Análisis de peor fold

Esto evita dispersión prematura.

---

## Verificaciones operativas (confirmadas)

### 1. Folds en manifest MVP-2
```
fold 0.0:  496 muestras
fold 1.0:  873 muestras
fold 2.0:  477 muestras
NaN:        73 muestras (HEK293 + excluidos, no usar)
```

**Decisión:** Usar folds 0/1/2, excluir NaN.

---

### 2. Subsets PRETRAIN/FINETUNE/EVAL (confirmados)

```
PRETRAIN: 715 muestras (cualquier ómica + PDL, todos treatments)
FINETUNE: 161 muestras (Control + Normal + PDL + alguna ómica)
EVAL:     146 muestras (finetune + 2+ modalidades)
```

**Archivos separados:**
- `manifest_mvp2_pretrain_20260328_143235.csv`
- `manifest_mvp2_finetune_20260328_143235.csv`
- `manifest_mvp2_eval_20260328_143235.csv`

---

### 3. Columnas join con matrices (confirmadas)

**RNA:**
- Columna: `rna_matrix_col`
- Formato: `Sample_21`, `Sample_22`, etc.
- Matriz: `mvp2_rna_selected_20260328_143235.csv.gz` (345 muestras × 2027 genes)

**Metilación:**
- Columna: `met_matrix_col`
- Formato: `203784950011_R01C01 beta`, `203784950064_R03C01 beta`, etc.
- Matriz: `mvp2_met_selected_20260328_143235.csv.gz` (479 muestras × 10000 CpGs)

**Biomarcadores:** Directamente en manifest (telomere_length, mtdna_cn, clock_*, etc.)

---

### 4. Rutas hardcoded (confirmadas)

```python
DATA_DIR = "/Users/JCB/Documentos/Proyecto Integrador/data/"
MANIFEST_DIR = "/Users/JCB/Documentos/Proyecto Integrador/data/manifests/"
RESULTS_DIR = "/Users/JCB/Documentos/Proyecto Integrador/results/"
```

---

### 5. Variables de dominio (confirmadas)

Disponible en manifest: `['study_part']`

Para batch probe:
- RNA/Met: usar `study_part`
- Si metadata GEO tiene batch técnico → incorporar después

---

## Protocolo final MVP-2 (tabla ejecutable)

| # | Experimento | Hipótesis | Input | Modelo | Métricas | Criterio éxito tentativo | Decisión siguiente |
|---|-------------|-----------|-------|--------|----------|-------------------------|-------------------|
| **1a** | Baseline RNA Elastic Net | Señal lineal PDL existe en HVGs | 2027 genes VST, folds 0/1/2 | Elastic Net CV (α) | Spearman, MAE, R² por fold, worst fold | Reportar como upper bound lineal | Avanzar a 1b |
| **1b** | Baseline RNA XGBoost | No-linealidad mejora sobre lineal | 2027 genes VST | XGBoost (max_depth, n_estimators tuning) | Spearman, MAE, R² por fold | Reportar como upper bound no-lineal | Si Spearman < 0.4 → señal débil, advertir |
| **1c** | Baseline Met Elastic Net crudo | Señal lineal existe en CpGs | 10K CpGs betas, folds 0/1/2 | Elastic Net CV | Spearman por fold, worst fold | Reportar | Avanzar a 1d |
| **1d** | Baseline Met PCA+Elastic Net | Señal comprimible en PCs | PCA(85% varianza) → Elastic Net | PCA + Elastic Net CV | Spearman, componentes PCA usados | Si ≈ 1c, señal es lineal gruesa | Avanzar a 1e (opcional) |
| **1e** | Baseline Met XGBoost (opcional) | No-linealidad útil en metilación | 10K CpGs o PCs | XGBoost | Spearman | **Solo si 1c/1d dejan incertidumbre sobre no-linealidad útil** | Fase 1 completa → Fase 2 |
| **2** | Encoder RNA MLP baseline | Deep learning captura señal ≥ baseline | 2027 genes → z_rna (256) → PDL_hat | MLP 2-capas, dropout 0.5, ranking sobre PDL_hat | Spearman, worst fold, ΔAUC(study_part), UMAP(z_rna) | Spearman ≥ 90% baseline EN, worst fold ≥ 0.4, ΔAUC < 0.2 | Si PASA → 3; Si FALLA → diagnosticar overfitting, considerar más dropout o pretrain |
| **3** | Encoder Met PCA+MLP | Deep sobre PCs captura señal | PCA(500-700) → z_met (256) → PDL_hat | MLP 2-capas, dropout 0.5 | Spearman, worst fold, ρ(z_met, clocks) parcial | Spearman ≥ 90% baseline PCA+EN, **ρ(clocks) > 0.3 coherente biológicamente, consistente en folds** | Si PASA → 4; Si FALLA → más regularización |
| **4** | Encoder Bio MLP | Relojes+telómero+mtDNA predicen PDL | 8-10 features escalares → z_bio (256) → PDL_hat | MLP 2-capas | Spearman (esperado alto) | Spearman > 0.6 (relojes deben funcionar) | Avanzar a 5 |
| **5a** | Cross-modal correlación | Embeddings alineados entre modalidades | z_rna, z_met, z_img en pares reales (RNA∩Img, Met∩Img, RNA∩Met) | Correlación Pearson/Spearman por dimensión | **ρ promedio, mediana, top-K dims, retrieval opcional** | ≥ 1 par con ρ_mediana > 0.15 | Si PASA → 5b; Si NO → diagnosticar encoders individuales |
| **5b** | Cross-modal CCA | Componentes canónicas capturan alineación | Pares RNA∩Img, Met∩Img | CCA (5-10 componentes) | Correlación canónica por componente, top component | **Top component > 0.3** | Si PASA → GO fusión (MVP-3); Si NO → NO-GO fusión |
| **5c** | Cross-modal correlación parcial | Señal compartida más allá de PDL | Residuos z vs PDL | Correlación parcial | ρ(z_rna, z_img \| PDL) | > 0.1 (evidencia complementariedad) | Documentar alineación |
| **6a** | (Opcional) RNA VAE | Regularización VAE ayuda vs overfitting | **Solo si exp 2 falla por overfitting brutal** | β-VAE 256-dim, supervisión PDL | Spearman, KL divergence, reconstrucción | Mejora sobre MLP baseline | Solo si tiempo sobra y hay justificación |
| **6b** | (Opcional) Met Sparse L1 | CpGs importantes interpretables | **Solo si exp 3 funciona bien** | MLP con L1 en capa entrada | Spearman, top-100 CpGs, overlap con clock sites | Interpretabilidad para tesis | Solo si tiempo sobra |

---

## Orden de ejecución (estricto)

### **Fase 1: Baselines clásicos (obligatorio, 1-2 días)**
1. Experimentos 1a → 1b → 1c → 1d
2. XGBoost Met (1e) solo si necesario
3. **Output:** Upper bounds de señal por modalidad

### **Fase 2: Encoder RNA (2-3 días)**
4. Experimento 2 (MLP baseline)
5. **Checkpoint:** Reporte completo (métricas, plots, batch probe) antes de avanzar

### **Fase 3: Encoder Met (2-3 días)**
6. Experimento 3 (PCA+MLP)
7. **Checkpoint:** Reporte completo antes de avanzar

### **Fase 4: Encoder Bio (1 día)**
8. Experimento 4 (trivial)

### **Fase 5: Análisis cross-modal (1-2 días)**
9. Experimentos 5a → 5b → 5c
10. **Decisión GO/NO-GO para MVP-3**

### **Fase 6: Opcionales (solo si justificado)**
11. Experimento 6a (VAE) solo si 2 falló
12. Experimento 6b (Sparse) solo si 3 funcionó y queda tiempo

**Regla de disciplina:** No abrir Fase 6 hasta que Fases 2-5 estén completas y documentadas.

---

## Estructura de resultados

```
results/
├── mvp2_baselines/
│   ├── elasticnet_rna_results.json
│   ├── elasticnet_met_crudo_results.json
│   ├── elasticnet_met_pca_results.json
│   ├── xgboost_rna_results.json
│   ├── xgboost_met_results.json (opcional)
│   ├── predictions_rna_fold{0,1,2}.csv
│   ├── predictions_met_fold{0,1,2}.csv
│   └── plots/
│       ├── scatter_rna_elasticnet.png
│       ├── scatter_met_pca_elasticnet.png
│       └── variance_explained_pca.png
│
├── mvp2_encoder_rna/
│   ├── fold0_model.pt
│   ├── fold1_model.pt
│   ├── fold2_model.pt
│   ├── embeddings_z_rna.csv
│   ├── metrics.json
│   ├── training_curves.png
│   └── plots/
│       ├── scatter_pdl_pred.png
│       ├── umap_z_rna_by_pdl.png
│       └── batch_probe_study_part.png
│
├── mvp2_encoder_met/
│   ├── pca_transformer.pkl
│   ├── fold{0,1,2}_model.pt
│   ├── embeddings_z_met.csv
│   ├── metrics.json
│   └── plots/
│       ├── correlation_with_clocks.png
│       └── umap_z_met.png
│
├── mvp2_encoder_bio/
│   ├── model.pt
│   ├── embeddings_z_bio.csv
│   └── metrics.json
│
├── mvp2_crossmodal/
│   ├── correlation_matrix.csv
│   ├── cca_results.json
│   ├── alignment_report.md
│   ├── decision_GO_NO_GO.txt
│   └── plots/
│       ├── heatmap_crossmodal_correlation.png
│       ├── cca_canonical_correlations.png
│       └── umap_joint_rna_img.png
│
└── mvp2_optional/
    ├── rna_vae/ (solo si aplica)
    └── met_sparse/ (solo si aplica)
```

---

## Criterios de avance a MVP-3 (fusión multimodal)

**Solo avanzar si se cumplen TODOS:**

✅ **Al menos 2 de 3 encoders ómicos (RNA, Met, Bio) cumplen:**
- Spearman ≥ 0.6 (o ≥ 90% del baseline clásico)
- Worst fold ≥ 0.5 (robustez mínima)
- ΔAUC(study_part) < 0.2 (contaminación tolerable)

✅ **Análisis cross-modal muestra alineación:**
- Al menos 1 par de modalidades con ρ_mediana > 0.15
- CCA top component > 0.3
- Correlación parcial (controlando PDL) > 0.1 en al menos 1 par

✅ **Embeddings tienen estructura interpretable:**
- UMAP muestra gradiente PDL visible
- No colapso a constante (varianza z > 0.01 en todas dims)

**Si alguno falla:**
- **NO avanzar a fusión**
- Diagnosticar: ¿encoder individual débil? ¿señal en baseline ya era baja? ¿overfitting?
- Iterar: más regularización, arquitectura más simple, o documentar limitación honestamente

---

## Métricas estándar por experimento

### Baselines clásicos (1a-1e)
- Spearman(PDL_pred, PDL_true) por fold
- MAE, R² por fold
- Worst fold Spearman
- Predictions CSV para análisis posterior

### Encoders profundos (2, 3, 4)
- Spearman(PDL_pred, PDL_true) por fold (promedio y worst)
- MAE, R² por fold
- **ΔAUC batch probe:** entrenar clasificador auxiliar `study_part` desde z, medir AUC - 0.5
- **Overfitting gap:** train vs val Spearman
- **Embedding quality:**
  - Varianza por dimensión (no colapso)
  - UMAP coloreado por PDL_bin, cell_line, study_part
- **Correlaciones auxiliares:**
  - z vs telómero, mtDNA (sanity check biológico)
  - z_met vs clocks (correlación parcial)

### Cross-modal (5a-5c)
- **Correlación por dimensión:** promedio, mediana, std
- **Top-K dimensiones** con mayor alineación
- **CCA:** correlación canónica por componente (top 5-10)
- **Correlación parcial:** ρ(z_rna, z_img | PDL)
- **Retrieval (opcional):** dado z_rna, recuperar z_img correcto en top-K vecinos (precisión)

---

## Limitaciones documentadas a priori

**Estas limitaciones son conocidas y se documentarán honestamente:**

- **Dataset pequeño:** 345 RNA, 479 Met → overfitting esperado, regularización crítica
- **Pairing limitado:** Solo 32 muestras con RNA∩Met∩Img → fusión triple-modal débil
- **Folds desbalanceados:** Fold 1 tiene 873 muestras, folds 0/2 ~490 → worst fold importa más
- **6 donantes sanos:** Generalización cross-donor limitada
- **Selección HVGs/CpGs sobre todo dataset:** Leak teórico menor (varianza, no target), aceptable pero documentar
- **Relojes epigenéticos como proxy:** No son ground truth de hallmarks, son correlatos

---

## Productos esperados de MVP-2

### Artefactos
1. **6 notebooks ejecutables** (5 núcleo + 1-2 opcionales)
2. **Modelos entrenados** (.pt por fold)
3. **Embeddings** (CSV con z_rna, z_met, z_bio por muestra)
4. **PCA transformer** (para inference Met)
5. **Métricas agregadas** (JSON por experimento)
6. **Plots diagnósticos** (scatter, UMAP, batch probe, CCA)

### Documentación
7. **Reporte MVP-2** (.docx):
   - Baselines vs deep learning
   - Métricas por modalidad
   - Análisis cross-modal
   - Decisión GO/NO-GO fusión
   - Limitaciones documentadas
8. **Estado proyecto actualizado** (.md)
9. **Tabla de métricas comparativas** (Elastic Net vs MLP por modalidad)

### Decisión final
10. **GO/NO-GO para MVP-3** (fusión multimodal)

---

## Notas finales

- **Criterios tentativos, no dogmas:** Los umbrales (0.6, 0.15, 0.3) son heurísticas razonables, sujetas a revisión
- **Honestidad metodológica:** Si algo falla, diagnosticar y documentar, no forzar
- **Disciplina experimental:** Completar cada fase antes de avanzar
- **Interpretabilidad gratis:** Si Elastic Net o SHAP dan insights, usarlos (más defendible que attention weights)
- **Fusión es consecuencia, no dogma:** Si cross-modal falla, documentar limitación honestamente

---

**Fin del plan MVP-2.**  
**Siguiente paso:** Generar `mvp2_02_baselines_clasicos.ipynb`
