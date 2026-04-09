# Diagnóstico Final Refinado — Baselines MVP-2

**Fecha:** 29 de marzo de 2026  
**Versión:** 3.0 (refinada con precisión metodológica)  
**Analista:** Claude (asistente técnico-científico)

---

## Prefacio metodológico

Los resultados de baselines indican que **RNA presenta una señal fuerte y mayormente lineal**, mientras que **metilación requiere reducción dimensional previa**. Sin embargo, la interpretación de los peores folds debe considerarse **provisional**, pues la composición por `pdl_norm`, `study_part` y condición clínica introduce desplazamientos de distribución que pueden reflejar tanto sesgo experimental como **verdadera extrapolación biológica**.

Este diagnóstico distingue entre:
- **Evidencia establecida:** Métricas de baselines, composición de folds
- **Inferencias plausibles:** Causas de worst fold, batch effects
- **Hipótesis abiertas:** Interpretación causal de ΔAUC, efecto de SURF1

---

## Resumen de métricas (evidencia establecida)

### RNA-seq (2027 genes, 339 muestras)

| Modelo | Spearman promedio | Worst fold | Desv. est. |
|--------|-------------------|------------|------------|
| Elastic Net | **0.901** | 0.847 | 0.039 |
| XGBoost | 0.852 | 0.708 | 0.102 |

**Folds:** 0 (111 val), 1 (181 val), 2 (47 val)

---

### Metilación (10K CpGs, 479 muestras)

| Modelo | Spearman promedio | Worst fold | Desv. est. |
|--------|-------------------|------------|------------|
| Elastic Net (crudo) | 0.712 | 0.527 | 0.162 |
| Elastic Net (PCA, 8 comp) | **0.772** | 0.610 | 0.117 |

**Folds:** 0 (143 val), 1 (196 val), 2 (140 val)

---

## Hallazgos establecidos (evidencia)

### 1. RNA: Señal fuerte y mayormente lineal

**Evidencia:**
- Elastic Net: 0.901 Spearman, σ=0.039
- XGBoost: 0.852 (peor que EN, no mejora)
- Elastic Net selecciona 43-133 genes (sparsity fuerte)

**Conclusión robusta:**
- Señal es **lineal** (no-linealidad no ayuda)
- Sparse (2-6% de genes necesarios)
- Upper bound clásico es 0.901

**Implicación:**
- MLP profundo **no tiene por qué superar EN en Spearman**
- Valor del encoder: **z_rna fusionable**, no competencia con EN
- Arquitectura simple (2 capas) es suficiente

---

### 2. Metilación: Reducción PCA obligatoria

**Evidencia:**
- Crudo: 0.712 Spearman, σ=0.162, R² negativos graves
- PCA: 0.772 Spearman, σ=0.117, solo 8 componentes
- Δ(PCA - Crudo) = +0.060

**Conclusión robusta:**
- PCA **mejora** señal (no destruye)
- 8 componentes capturan 85% varianza
- Reduce variabilidad entre folds

**Implicación:**
- Pipeline encoder Met: **PCA → MLP** es obligatorio
- PCA a 500-700 componentes antes del encoder

---

### 3. Composición por fold NO es homogénea

**Evidencia establecida:**

| Variable | Fold 0 | Fold 1 | Fold 2 |
|----------|--------|--------|--------|
| PDL mean (RNA) | 0.412 | 0.465 | **0.560** |
| Study_part dominante | 2 | 2 | **1** |
| SURF1 en Met | 0% | 13% | 11% |

**Conclusión robusta:**
- Fold 2 tiene **PDL más alto** (muestras más envejecidas)
- Study_part **no está balanceado** entre folds
- SURF1 distribuido, no concentrado

**Implicación:**
- Worst fold refleja **distribución distinta**, no solo error del modelo
- GroupKFold prioriza anti-leakage sobre balance → trade-off metodológico

---

## Inferencias plausibles (NO establecidas causalmente)

### Inferencia 1: Fold 2 peor por distribución PDL + study_part

**Observación:**
- Fold 2 RNA: PDL mean 0.560 (muestras viejas)
- Fold 2 RNA: dominado por study_part 1 (80/140)
- Fold 2 RNA: Spearman EN 0.847, XGB 0.708

**Inferencia plausible:**
- Encoder entrenado en muestras jóvenes (Folds 0+1, PDL ~0.44)
- Al evaluar en muestras viejas (Fold 2, PDL 0.56), generaliza peor
- Study_part 1 puede tener protocolo distinto → batch effect

**Advertencia metodológica:**
- Esta es **asociación descriptiva**, no causalidad probada
- Faltan análisis estratificados:
  - Baseline usando solo `pdl_norm` + `study_part`
  - Desempeño dentro de bins de PDL
  - Batch probe **condicional** (controlar PDL + cell_line)

**Uso correcto:**
- **NO afirmar:** "Fold 2 colapsa por PDL + study_part" (causal)
- **SÍ afirmar:** "Fold 2 tiene PDL distinto y study_part diferente, lo cual **puede contribuir** al peor desempeño" (asociación)

---

### Inferencia 2: Fold 0 Met mejor por ausencia de SURF1

**Observación:**
- Fold 0 Met: 100% Normal (sin SURF1)
- Fold 0 Met: Spearman 0.882 (mejor)
- Folds 1/2: 87-89% Normal, 11-13% SURF1, Spearman 0.824/0.610

**Inferencia plausible:**
- SURF1 mutation puede introducir ruido en señal de aging
- Fold 0 sin SURF1 tiene señal más limpia

**Advertencia metodológica:**
- **Plausible, no probado**
- Faltaría: Comparar Normal-only en Folds 1/2 vs Fold 0
- Fold 2 peor puede ser por PDL, no solo SURF1

**Uso correcto:**
- **NO afirmar:** "Fold 0 es mejor porque no tiene SURF1" (causal)
- **SÍ afirmar:** "Fold 0 carece de SURF1 y tiene mejor desempeño, lo cual **sugiere** que la señal de aging es más limpia sin mutación" (hipótesis)

---

### Inferencia 3: Study_part como batch effect

**Observación:**
- Study_part desbalanceado entre folds
- Study_part está acoplado a protocolo experimental

**Inferencia plausible:**
- Encoder puede aprender protocolo en vez de aging
- ΔAUC(study_part) batch probe cuantificará contaminación

**Advertencia metodológica crítica:**
- En MVP-1 ya viste que **dominio y biología están enredados**
- Variables de adquisición están acopladas a PDL, cell_line
- Adversarial ciego (DANN) puede **borrar señal útil**

**Regla corregida para batch probe:**

❌ **Incorrecto (receta automática):**
> "Si ΔAUC(study_part) > 0.2 → usar DANN"

✅ **Correcto (análisis condicional):**
1. Medir ΔAUC(study_part) desde z_rna
2. **Investigar si predictibilidad persiste después de controlar:**
   - PDL (correlación parcial)
   - Cell_line (stratificación)
   - Clinical_condition
3. Si porción residual es alta → batch effect probable
4. Considerar mitigación suave (CORAL, MMD) **no DANN ciego**
5. Si ΔAUC < 0.15 → tolerable, documentar

**Uso correcto:**
- **NO proponer:** Umbral automático ΔAUC → DANN
- **SÍ proponer:** ΔAUC como diagnóstico, seguido de análisis condicional

---

## Lectura correcta de worst fold

### ⚠️ Corrección importante

**Lectura incorrecta (excusa metodológica):**
> "Fold 2 débil es problema del split, no del modelo"

**Lectura correcta (honestidad científica):**
> "Fold 2 es **test de extrapolación biológica y de dominio**"

**Argumentación:**
- Fold 2 tiene muestras más envejecidas (PDL alto)
- Fold 2 tiene protocolo distinto (study_part 1)
- Si modelo colapsa en Fold 2, **no basta** decir "el split estaba sesgado"
- Hay que aceptar que la representación **no generaliza bien a esa región del manifold**

**Implicación:**
- Worst fold < target **NO es solo defecto del split**
- Es señal de que el modelo aún **no extrapolará bien** a:
  - Estados de envejecimiento tardíos
  - Dominios experimentales menos vistos
- **Documentar como limitación científica**, no solo metodológica

**Defensa correcta en tesis:**
1. ✅ GroupKFold prioriza anti-leakage (justificado)
2. ✅ Fold 2 tiene distribución distinta (documentado)
3. ⚠️ **Pero:** Modelo aún no generaliza a esa distribución (limitación)
4. ✅ Trabajo futuro: aumentar datos en región de PDL alto, o domain adaptation

---

## Decisiones de avance (refinadas)

### ✅ **AVANZAR a Encoder RNA (Experimento 2)**

**Justificación:**
- Señal fuerte (0.901)
- Upper bound clásico claro
- Problemas de folds identificados y documentables

**Target refinado:**

| Métrica | Umbral tentativo | Justificación |
|---------|------------------|---------------|
| Spearman promedio | ≥ 0.811 | 90% de EN (0.901) |
| Worst fold | ≥ 0.76 | 90% de EN worst fold (0.847) |
| ΔAUC(study_part) bruto | < 0.20 | Diagnóstico inicial |
| ΔAUC(study_part) condicional | < 0.10 | Después de controlar PDL + cell_line |

**Expectativa realista:**
- MLP **puede igualar** EN (0.9), **no garantizado**
- Worst fold probablemente < 0.76 (extrapolación a PDL alto)
- **Documentar worst fold como extrapolación biológica**, no solo sesgo de split
- Valor del encoder: **z_rna fusionable**, no Spearman

**Arquitectura final:**
```python
RNA (2027) → Dropout(0.5) → Linear(512) → ReLU → Dropout(0.3)
           → Linear(256) → z_rna
           → Linear(1) → PDL_hat

Loss = MSE(PDL, PDL_hat) + λ_rank * RankingLoss(PDL_hat, PDL)
```

**Diagnósticos obligatorios:**
1. ✅ Spearman por fold
2. ✅ ΔAUC(study_part) bruto
3. ✅ ΔAUC(study_part) **condicional** (residual después de PDL + cell_line)
4. ✅ Overfitting gap (train-val Spearman)
5. ✅ UMAP(z_rna) coloreado por PDL bins
6. ✅ Correlación parcial z_rna vs biomarcadores

---

### ✅ **AVANZAR a Encoder Met (Experimento 3) CON PRECAUCIÓN**

**Justificación:**
- Señal moderada (0.772)
- PCA obligatorio
- Worst fold 0.610 es **límite de aceptabilidad**

**Target refinado:**

| Métrica | Umbral tentativo | Justificación |
|---------|------------------|---------------|
| Spearman promedio | ≥ 0.695 | 90% de PCA+EN (0.772) |
| Worst fold | ≥ 0.55 | 90% de PCA worst fold (0.610) |
| Correlación con clocks | ρ > 0.3 | Sanity check biológico |
| R² Fold 0/1 | > 0 | Calibrado en mejores folds |

**Expectativa realista:**
- MLP con PCA puede **igualar** 0.77
- Fold 2 seguirá débil (0.55-0.61) → **extrapolación a PDL alto**
- R² negativos en Fold 2 son **tolerables** si Spearman funciona

**Pipeline final:**
```python
Met (10K) → StandardScaler → PCA(n_components=0.85)
          → Dropout(0.5) → Linear(256) → ReLU → Dropout(0.3)
          → Linear(256) → z_met
          → Linear(1) → PDL_hat
```

---

## Hipótesis abiertas (requieren más análisis)

Las siguientes preguntas **NO están resueltas** con los baselines actuales:

### Hipótesis 1: ¿Study_part es batch puro o está acoplado a biología?

**Para verificar:**
- Regresión logística: `study_part ~ pdl_norm + cell_line + clinical_condition`
- Si AUC > 0.8 después de controlar → acoplado a biología
- Si AUC < 0.6 después de controlar → batch técnico separable

**Implicación:**
- Si acoplado → DANN es peligroso
- Si separable → mitigación suave es viable

---

### Hipótesis 2: ¿Fold 0 Met mejor por SURF1 o por PDL distribution?

**Para verificar:**
- Comparar Normal-only en Folds 0 vs 1/2
- Estratificar por bins de PDL dentro de cada fold
- Si Normal-only Fold 0 sigue siendo mejor → SURF1 sí afecta
- Si diferencia desaparece → PDL distribution es causa

---

### Hipótesis 3: ¿Cuáles genes selecciona Elastic Net?

**Para verificar:**
- Entrenar EN en todos los datos RNA
- Extraer top 50 genes por |coeficiente|
- Verificar overlap con 40 hallmark genes forzados
- Si overlap alto → señal biológica
- Si genes desconocidos → investigar vías

**Valor:** Interpretabilidad biológica para tesis

---

## Limitaciones documentadas (versión refinada)

### Limitación 1: Distribución PDL no homogénea entre folds

**Evidencia:**
- Fold 2 RNA: PDL mean 0.560 vs 0.412/0.465 otros folds

**Defensa metodológica:**
- GroupKFold por cell_line prioriza **anti-leakage**
- Estratificación por PDL bins rompería independencia cell_line
- Trade-off metodológico justificado

**Defensa científica (honesta):**
- Fold 2 es **test de extrapolación** a muestras más envejecidas
- Modelo aún no generaliza bien a esa región
- **Limitación del modelo y del dataset**, no solo del split
- Trabajo futuro: más datos en PDL alto, domain adaptation

---

### Limitación 2: Study_part acoplado a folds

**Evidencia:**
- Fold 2 dominado por study_part 1
- Folds 0/1 dominados por study_part 2

**Defensa:**
- Study_part es covariable de protocolo, **puede estar acoplada a biología**
- Batch probe ΔAUC cuantifica contaminación
- Análisis **condicional** (controlando PDL) determina si es batch puro
- Si ΔAUC condicional < 0.1 → tolerable

---

### Limitación 3: R² negativos en metilación

**Evidencia:**
- Met crudo: R² hasta -5.197
- Met PCA Fold 2: R² -0.047

**Defensa:**
- **Spearman es la métrica principal** (ranking, no magnitud)
- R² mide calibración de magnitud
- Para aging, **ranking > predicción absoluta PDL**
- PDL no es medición física precisa, es proxy
- R² negativo marginal (-0.047) con Spearman 0.610 es **utilizable**

---

### Limitación 4: Folds desbalanceados

**Evidencia:**
- RNA Fold 2: 47 muestras (14%)
- RNA Fold 1: 181 muestras (53%)

**Defensa:**
- Consecuencia de GroupKFold por cell_line
- 6 donantes sanos → balance perfecto es imposible
- Priorizamos anti-leakage sobre balance
- **Dataset pequeño, no error metodológico**

---

## Acciones antes de Experimento 2

### ✅ Completadas

1. ✓ Baselines clásicos (EN, XGB)
2. ✓ Composición por fold (cell_line, PDL, study_part, clinical_condition)
3. ✓ Diagnóstico de causas plausibles worst fold

### 🔶 Pendientes (opcionales, no bloqueantes)

4. ⚪ Top genes Elastic Net (interpretabilidad)
5. ⚪ Visualización PDL distribution por fold (figura tesis)
6. ⚪ Análisis estratificado Normal-only (verificar hipótesis SURF1)
7. ⚪ Baseline `pdl_norm + study_part` (cuantificar asociación)

---

## Siguiente paso

**Generar Notebook 2:** `mvp2_03_encoder_rna_baseline.ipynb`

Con:
- MLP simple (2 capas), dropout 0.5/0.3
- RankingLoss sobre PDL_hat
- Batch probe ΔAUC(study_part) bruto **y condicional**
- PRETRAIN → FINETUNE → EVAL
- Early stopping patience 20
- Target: Spearman ≥ 0.811, ΔAUC condicional < 0.10

**¿Procedo con Notebook 2 o ejecutamos primero alguna de las acciones pendientes?**

---

## Resumen ejecutivo (una línea)

**RNA tiene señal fuerte y lineal (EN 0.901); Met requiere PCA (0.772); encoders profundos valen por z fusionable, no por superar EN; worst fold refleja extrapolación biológica, no solo sesgo de split.**

---

**Fin del diagnóstico refinado.**
