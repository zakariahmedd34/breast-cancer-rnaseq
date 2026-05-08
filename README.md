# Breast Cancer ER Status Classification from mRNA Expression

Predicts **Estrogen Receptor (ER) status** (ER+ vs ER-) in breast cancer tumors using gene expression data. Built as a presentation-ready pipeline for biotech audiences.

---

## Problem

Breast cancer has two major subtypes:

- **ER-positive** — driven by estrogen, responds to hormone therapy (tamoxifen)
- **ER-negative** — does not respond to hormone therapy, requires chemotherapy

Knowing a patient's ER status is critical for choosing the right treatment. This pipeline demonstrates how mRNA expression data alone can classify ER status using machine learning.

---

## Dataset

| File | Description |
|---|---|
| `data/data_mrna_seq_v2_rsem_zscores_ref_all_samples.txt` | mRNA expression z-scores — 123 tumor samples × 16,224 genes |
| `data/data_clinical_patient.txt` | Clinical metadata — 55 patients, 36 features (ER status, survival, etc.) |

Source: [cBioPortal](https://www.cbioportal.org) — Aurora Breast Cancer cohort (AURORA-VICC).

> **Note:** The mRNA sample IDs and clinical patient IDs use different naming systems and cannot be directly joined without a mapping file. The pipeline uses **ESR1 gene expression** as a biologically valid proxy for ER status (ESR1 encodes the Estrogen Receptor protein directly).

See [`data/DATA_DESCRIPTION.md`](data/DATA_DESCRIPTION.md) for full column descriptions.

---

## Pipeline

```
16,224 genes
     │
     ▼
Label creation      ESR1 median split → ER+ (label=1) / ER- (label=0)
     │
     ▼
Train/Test split    75% train (92 samples) / 25% test (31 samples), stratified
     │
     ▼
4 Model Pipelines   SelectKBest(f_classif, k=50) → Scaler → Classifier
     │
     ▼
Evaluation          Accuracy, Recall, F1 Score, ROC AUC — train AND test
     │
     ▼
Gene Ranking        Top 20 genes by RF importance + LR |coefficient|
```

---

## Models

| Model | Type | Key Parameters |
|---|---|---|
| Logistic Regression | Linear | C=0.1 |
| Linear SVM | Linear | C=0.1, kernel=linear |
| Random Forest | Non-linear | n_estimators=100, max_depth=5, min_samples_leaf=3 |
| KNN | Non-linear | k=5 |

All models use `SelectKBest(f_classif, k=50)` as the first pipeline step — the ANOVA F-test selects the 50 genes most differentially expressed between ER+ and ER- **on training data only**, preventing data leakage.

---

## Notebook

[`notebook/er_status_prediction_from_mrna.ipynb`](notebook/er_status_prediction_from_mrna.ipynb)

The notebook is structured as a presentation — every code block is paired with a markdown cell explaining the biology and the ML concept in plain language for a biotech audience.

**Sections:**
1. Data loading and inspection
2. Label creation from ESR1 expression
3. EDA — ESR1 distribution + class balance
4. Feature selection rationale (data leakage explanation)
5. Model definitions (4 pipelines)
6. Training
7. Evaluation metrics — train and test
8. Confusion matrices
9. ROC AUC comparison (bar chart)
10. Learning curves (F1 score)
11. Gene ranking — RF importance + LR coefficients

---

## Setup

```bash
pip install pandas numpy matplotlib scikit-learn jupyter
jupyter notebook notebook/er_status_prediction_from_mrna.ipynb
```

**Python:** 3.8+  
**Key packages:** scikit-learn, pandas, numpy, matplotlib

---

## Key Design Decisions

**Why ESR1 as label?**
ESR1 directly encodes the Estrogen Receptor. High ESR1 z-score = ER+ tumor. This avoids needing the clinical file and is biologically grounded.

**Why SelectKBest inside the pipeline?**
Computing gene variance (or any ranking criterion) on the full dataset before the train/test split leaks test information into feature selection, inflating evaluation metrics. Putting `SelectKBest` inside the pipeline ensures it only sees training data.

**Why f_classif and not variance?**
`f_classif` uses the ANOVA F-test to select genes based on how differently expressed they are between ER+ and ER-. Variance selects genes that vary a lot, regardless of whether that variation correlates with the label.

**Why C=0.1 for LR and SVM?**
With only 92 training samples and 50 features, strong regularization prevents the linear models from overfitting to individual genes.
