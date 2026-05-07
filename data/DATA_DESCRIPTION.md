# Data Description

## Source

**Aurora Breast Cancer Cohort (AURORA-VICC)**  
Downloaded from [cBioPortal](https://www.cbioportal.org).

---

## File 1 — mRNA Expression

**File:** `data_mrna_seq_v2_rsem_zscores_ref_all_samples.txt`  
**Format:** Tab-separated, rows = genes, columns = samples  
**Size:** 16,224 genes × 123 tumor samples

### What the values mean

Values are **z-scores** computed relative to all samples in the dataset:

```
z = (sample_expression - mean_across_all_samples) / std_across_all_samples
```

| z-score | Meaning |
|---|---|
| z ≈ 0 | Average expression — gene expressed at typical level |
| z > 1.5 | Overexpressed — gene is highly active in this tumor |
| z < -1.5 | Underexpressed — gene is silenced or suppressed in this tumor |

### Columns

| Column | Type | Description |
|---|---|---|
| `Hugo_Symbol` | string (index) | Official gene symbol (e.g. ESR1, BRCA1) |
| `Entrez_Gene_Id` | integer | NCBI Entrez database gene ID — dropped during loading |
| `AUR_01_20_01` ... | float | Z-score expression for each tumor sample |

### After loading and transposing

The notebook transposes the file so rows = samples and columns = genes, which is the format expected by scikit-learn:

```
Shape after transpose: 123 samples × 16,224 genes
```

### Key gene used as label

| Gene | Role |
|---|---|
| **ESR1** | Encodes the Estrogen Receptor alpha protein. High expression = ER+ tumor. Used as the classification label (median split). |

---

## File 2 — Clinical Patient Data

**File:** `data_clinical_patient.txt`  
**Format:** Tab-separated, 4 metadata header rows (starting with `#`), actual headers on row 5  
**Size:** 55 patients × 36 features

### Loading note

The file has 4 comment rows before the real header:

```python
clinical = pd.read_csv("data_clinical_patient.txt", sep="\t", skiprows=4)
```

### Key columns

| Column | Description |
|---|---|
| `PATIENT_ID` | Patient identifier (e.g. `AUR-AD9E`) |
| `ER_STATUS` | Estrogen Receptor status — Positive / Negative |
| `PR_STATUS` | Progesterone Receptor status |
| `HER2_STATUS` | HER2 receptor status |
| `SUBTYPE` | Molecular subtype (Luminal A, Luminal B, HER2-enriched, Basal-like) |
| `AGE` | Age at diagnosis |
| `OS_STATUS` | Overall survival status (0=alive, 1=deceased) |
| `OS_MONTHS` | Months of follow-up |
| `TUMOR_STAGE` | TNM tumor stage |

### Why the clinical file is not used for the label

The `PATIENT_ID` values in the clinical file (e.g. `AUR-AD9E`) do not match the sample IDs in the mRNA file (e.g. `AUR_01_20_01`). These are different identifier systems, and no mapping file was provided to link them. As a result, `ER_STATUS` from the clinical file cannot be joined to the mRNA expression data.

**Workaround:** The pipeline uses `ESR1` gene expression (median split) as a biologically equivalent proxy for ER status. This is valid because ESR1 directly encodes the Estrogen Receptor protein — high ESR1 expression is the molecular mechanism behind ER positivity.

---

## ID Mismatch Diagram

```
mRNA file sample IDs:        Clinical file patient IDs:
AUR_01_20_01                 AUR-AD9E
AUR_01_20_06                 AUR-BF12
AUR_01_20_04       ✗ no join AUR-CD34
...                          ...

→ Use ESR1 gene in mRNA file as the label instead
```

---

## Class Balance

After ESR1 median split on 123 samples:

| Class | Count | % |
|---|---|---|
| ER+ (label=1, ESR1 > median) | 61 | 49.6% |
| ER- (label=0, ESR1 ≤ median) | 62 | 50.4% |

The dataset is nearly perfectly balanced — no class weighting needed.
