# Proteins Linked to Learning in a Down Syndrome Mouse Model

*A re-analysis of the Mice Protein Expression dataset with mouse-level statistics and machine learning*

## 1. Introduction

Down syndrome is caused by an extra copy of chromosome 21 and leads to learning difficulties. The Ts65Dn mouse carries an extra copy of many of the matching genes and is a standard model of Down syndrome. These mice fail a learning task (context fear conditioning), but the drug **memantine**, which blocks NMDA receptors, can rescue their learning.

Higuera et al. (2015) measured 77 proteins in the cortex of these mice and used Self-Organizing Maps to find proteins linked to learning. This project asks three questions:

1. Which proteins change when **normal mice learn**?
2. Which proteins differ when **learning fails** in Down syndrome mice?
3. Which proteins does **memantine** change when it rescues learning?

The main improvement over a standard analysis is that every test and model works **at the level of the mouse**, not the row.

## 2. Data

- **72 mice** (38 control, 34 Ts65Dn), **77 proteins** measured in the nuclear fraction of the cortex.
- **8 classes** from three factors: genotype (control / Ts65Dn), behaviour (learning C/S / no-learning S/C) and treatment (memantine / saline). 7–10 mice per class.
- **15 technical replicates per mouse**: one brain sample was measured 15 times (3 replicates × 5 dilutions), giving 1,080 rows.

The dataset description states that each measurement "can be considered as an independent sample". This project shows that this assumption strongly inflates results.

## 3. Data cleaning

- **Duplicate column:** `ARC_N` and `pS6_N` are identical in all 1,080 rows. They are different proteins, so this is a data error; `pS6_N` was removed (76 proteins left).
- **Missing values come in whole mice:** when a protein is missing, it is usually missing for all 15 replicates (e.g. BCL2 was never measured in 19 of 72 mice). Gaps are spread across all 8 classes.

![Missing values by mouse](figures/missing_by_mouse.png)

- 3 almost-empty rows were removed (no mouse lost). Small gaps were filled with the same mouse's average. Remaining gaps were filled **inside the model pipeline** (`KNNImputer`), learning only from training mice, to avoid data leakage.

## 4. Exploratory analysis

- **Biology check:** proteins of chromosome 21 genes are higher in Ts65Dn mice (APP +17%, ITSN1 +15%, DYRK1A +11%, SOD1 +5%), as expected from the extra gene copy.

![Chromosome 21 proteins](figures/chr21_proteins.png)

- **Learning dominates:** SOD1 is ~2.3× higher in no-learning mice, with no overlap between groups, so a single protein separates learning from no-learning.

- **Outliers:** mouse 3484 has very high MAPK pathway proteins (DYRK1A, BRAF, pERK ~4.5× its class). Mice 293 and 3516 have very high AcetylH3K9. All mice were kept; models use robust scaling.
- **Correlations:** 14 protein pairs have |r| > 0.9, mostly proteins of the same pathway (BRAF → MEK → ERK) or a protein and its active form (NR1 / pNR1).

## 5. Statistical tests

**Method:** Mann–Whitney U test on **mouse averages** (one value per mouse), with Benjamini–Hochberg FDR correction for 76 proteins (significant: q < 0.05).

| Comparison | Mice | p < 0.05 | q < 0.05 |
|---|---|---|---|
| Learning, normal mice (c-SC-s vs c-CS-s) | 9 vs 9 | 27 | **17** |
| Failed learning (c-CS-s vs t-CS-s) | 9 vs 7 | 13 | **0** |
| Rescue by memantine (t-CS-s vs t-CS-m) | 7 vs 9 | 5 | **0** |

- Learning changes many proteins; the strongest are SOD1, pERK, ARC (a known memory gene) and IL1B.
- Failed learning and rescue show candidate proteins (NR2A, P38, Tau, APP; NUMB, SNCA), but none survive FDR correction with only 7–9 mice per group.
- **Pseudoreplication:** treating the 1,077 rows as independent mice would report 41 significant proteins for failed learning instead of 0.

![Pseudoreplication](figures/pseudoreplication.png)

- **Possible rescue:** P38 and Tau move away from normal in failed learning and back towards normal with memantine. APP stays high, as memantine cannot remove the extra gene copy.

![Normal, failed and rescued learning](figures/normal_failed_rescued.png)

## 6. Machine learning

**Method:** predict the class (8 groups) from the 76 proteins. Pipeline: `KNNImputer` → `RobustScaler` → model. Validation with `StratifiedGroupKFold` (5 folds), so each mouse is wholly in training or in testing.

| Model | Split by mouse (correct) | Random rows (wrong) |
|---|---|---|
| Baseline (most common class) | 14% | 14% |
| Logistic Regression | 66% | 99% |
| Random Forest | 59% | 100% |
| **SVM** | **69%** | 100% |

- With a random row split, the model recognises mice it has already seen, giving a misleading ~100%.
- Letting each mouse's rows vote raises the SVM accuracy to **76% per mouse** (55 of 72). Tuning the SVM (nested cross-validation) gave only a small gain (70%).
- **Failed learning (t-CS-s)** is the hardest class: 3 of 7 mice are predicted as rescued (t-CS-m).

![Confusion matrix](figures/confusion_matrix.png)

- Yes/no tasks: learning vs no learning **~100%**, Down syndrome vs normal **85%**, memantine vs saline **79%**.

## 7. Which proteins matter?

**Method:** permutation importance on the test mice of each fold.

![Feature importance](figures/feature_importance.png)

- The model relies most on **APP, ITSN1 and DYRK1A**, all chromosome 21 genes, followed by **Tau** and **NUMB** (both rescue candidates in Section 5).
- SOD1 ranks #1 in the statistics but only #64 for the model: its learning information is shared with many other proteins, so shuffling it alone barely matters. Permutation importance underestimates proteins with correlated "backups".

## 8. Limitations

- **Small sample:** 7–10 mice per class limits statistical power; small effects cannot be confirmed.
- **Technical replicates:** averaging them loses within-mouse variation; mixed models could use all rows correctly.
- **Correlated proteins** make single-protein importance hard to interpret.
- **Outliers** (mice 3484, 293, 3516) were kept; their cause (biology or measurement) is unknown.
- No independent test dataset was available.

## 9. Conclusions

1. The 15 replicates per mouse are **not independent**. Ignoring this inflates accuracy from 69% to ~100% and significant proteins from 0 to 41.
2. **Learning** produces the strongest protein changes (17 proteins, led by SOD1, pERK and ARC).
3. **Chromosome 21 proteins** (APP, ITSN1, DYRK1A) are higher in Down syndrome mice and are the model's most important features.
4. Failed and rescued learning are hard to separate. **P38, Tau and NUMB** are candidates for the memantine rescue but need confirmation in larger studies.
5. The dataset contains a **duplicated column** (ARC_N = pS6_N) that should be removed in any analysis.

## References

- Higuera C, Gardiner KJ, Cios KJ (2015). Self-Organizing Feature Maps Identify Proteins Critical to Learning in a Mouse Model of Down Syndrome. *PLoS ONE* 10(6): e0129126. https://doi.org/10.1371/journal.pone.0129126
- Higuera C, Gardiner K, Cios K (2015). *Mice Protein Expression* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C50S3Z
