# NSL-KDD vs UNSW-NB15: Feature Selection for Network Intrusion Detection

A side-by-side comparison of two intrusion-detection benchmarks. For each one the same pipeline is run: preprocessing, four feature-selection strategies, and three classifiers. The question is whether feature selection helps, and whether the answer depends on the dataset.

## Pipeline

```
raw CSV/TXT ─► clean ─► encode + scale ─► feature selection ─► classifiers ─► evaluation
               │          │                 │                    │
               │          │                 ├─ Full (baseline)   ├─ Random Forest
               │          │                 ├─ Filter: MI top-30 ├─ Linear SVM
               │          │                 │   + Pearson > 0.90 └─ XGBoost
               │          │                 ├─ Wrapper: RFECV
               │          │                 └─ Wrapper: SFS (15)
               │          └─ one-hot (categoricals), MinMaxScaler
               └─ drop duplicates and redundant / ID columns
```

## Datasets

| | NSL-KDD | UNSW-NB15 |
|---|---|---|
| Train / test rows | 25,192 (20% subset) / 22,544 | 107,740 / 55,945 (after de-duplication, from 175,341 / 82,332) |
| Features after one-hot | 117 | 194 |
| Target | 5 groups: Normal, DoS, Probe, R2L, U2R | 10 classes: Normal + 9 attack categories |
| Source | Canadian Institute for Cybersecurity | UNSW Canberra Cyber Range Lab |

Raw data is not committed. 

## Method notes

- **Filter:** mutual information ranking, then drop one of each feature pair with Pearson |r| > 0.90, then keep the top 30.
- **Wrapper 1, RFECV:** Random Forest estimator, 3-fold stratified CV, weighted F1.
- **Wrapper 2, SFS:** forward selection down to 15 features.
- **Why mutual information:** attacks rarely follow straight-line patterns, and MI can pick up non-linear relationships that a linear correlation score would miss.
- **UNSW-NB15 `service`:** the `-` placeholder (no service recorded) is kept as its own category, and `service` is one-hot encoded with `proto` and `state`.
- **Tuning:** `GridSearchCV` (5-fold stratified, weighted F1) for XGBoost and Random Forest, and a `C` sweep for the Linear SVM.
- Metrics: accuracy, weighted F1 and macro F1. Macro F1 matters most here because the rare classes (R2L, U2R, Worms, Shellcode) are the ones that count in security.

## Results

Default-hyperparameter runs, evaluated on each dataset's official test split.

**XGBoost** (best model on both datasets)

| Feature set | NSL-KDD acc / macro-F1 | UNSW-NB15 acc / macro-F1 |
|---|---|---|
| Full | 0.775 / 0.525 | 0.747 / 0.519 |
| Filter (MI top-30) | 0.766 / 0.507 | 0.747 / 0.526 |
| RFECV | 0.769 / 0.523 (K=37) | 0.747 / 0.537 (K=44) |
| SFS (K=15) | 0.760 / 0.507 | 0.726 / 0.494 |

**Random Forest**

| Feature set | NSL-KDD acc / macro-F1 | UNSW-NB15 acc / macro-F1 |
|---|---|---|
| Full | 0.749 / 0.515 | 0.635 / 0.468 |
| Filter (MI top-30) | 0.738 / 0.476 | 0.643 / 0.524 |
| RFECV | 0.745 / 0.502 (K=37) | 0.643 / 0.504 (K=44) |
| SFS (K=15) | 0.758 / 0.519 | 0.631 / 0.512 |

**Linear SVM** is the weakest model on both datasets (macro-F1 about 0.33 on UNSW-NB15 and 0.43–0.50 on NSL-KDD). It scores 0.00 on NSL-KDD's R2L class in the tuned runs.

### Observations

1. **XGBoost wins on both datasets**, and its accuracy changes little across feature sets. Fifteen to thirty features give up only a point or two against all 117 or 194.
2. **Feature selection helped more on UNSW-NB15 than on NSL-KDD.** With Random Forest on UNSW-NB15, macro-F1 rose from 0.468 on the full set to 0.524 with the MI filter. On NSL-KDD, the full set was as good as or better than the filters.
3. **Different features matter.** NSL-KDD's SFS picks lean on connection-level and service features (`src_bytes`, `count`, `flag_SF`, `service_*`). UNSW-NB15's picks lean on flow-level and TTL features (`sbytes`, `sttl`, `ct_state_ttl`, `smean`).
4. **Rare classes remain the problem.** NSL-KDD's R2L and U2R recall is very low even when overall accuracy looks fine.

### Caveats

- The two datasets don't share a label space (5 vs 10 classes), so the absolute numbers are not directly comparable. The comparison is about how the effect of feature selection changes, not which dataset is "easier".
- NSL-KDD uses the 20% training subset, and its test set contains attack types absent from training, which lowers accuracy.
- Models are evaluated on the official test split. The grid searches run cross-validation on train only.
- Results come from a single random seed (42), without repeated runs or confidence intervals.

## Repository layout

```
├── README.md
├── requirements.txt
├── notebooks/
│   ├── 01_nsl_kdd.ipynb           # NSL-KDD preprocessing, selection, models
│   └── 02_unsw_nb15.ipynb         # UNSW-NB15 preprocessing, selection, models
```

## Reproducing

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# download the datasets into data/ , then:
jupyter lab
```

RFECV and SFS are slow. Expect several minutes per dataset, and more on UNSW-NB15.

## References

- Tavallaee et al., "A detailed analysis of the KDD CUP 99 data set," 2009 (NSL-KDD).
- Moustafa & Slay, "UNSW-NB15: a comprehensive data set for network intrusion detection systems," 2015.
