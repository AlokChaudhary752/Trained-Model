# Trained-Model

Machine-learning experiments for **network traffic security**, built with Python, scikit-learn and Google Colab. The repository contains two related workstreams:

1. **DNS-over-HTTPS (DoH) detection** – binary classification (Benign vs. Malicious) of DoH flows using the *BCCC-CIRA-CIC-DoHBrw-2020* dataset.
2. **Multi-class network intrusion detection** – a Random Forest pipeline on flow-level data covering DoS, brute-force, SQL-injection and infiltration attacks (file names follow the CSE-CIC-IDS2018 capture days, Feb 14 – Mar 1, 2018).

> **Status:** work in progress. Data preparation and exploration are complete; model training and evaluation are only partly done (see [Results](#results) and [Roadmap](#roadmap)).

---

## Repository contents

| File | What it does |
|---|---|
| `detectionmodel.ipynb` | **Main DoH notebook.** EDA, outlier capping, label encoding, correlation-based feature pruning, Random Forest feature importance, stratified split, RobustScaler, and a 200-tree Random Forest. |
| `Untitled28.ipynb` | DoH exploratory analysis and preprocessing experiments (distributions, boxplots by label, correlation heatmap, Standard/MinMax scaling). |
| `Untitled32.ipynb` | DoH cleaning and visualisation: column-name cleanup, label mapping, max-abs normalisation, scatter plots and rate analysis per class. Also imports KNN for later use. |
| `Untitled36.ipynb` | **Intrusion-detection data pipeline.** Merges the attack-day CSVs, cleans them, engineers features and writes the train/test splits. |
| `Untitled37.ipynb` | Reloads the saved splits from Drive and sets up a Random Forest (100 trees). Training is not run yet. |

All notebooks are Colab notebooks (each has an *Open in Colab* badge at the top) and read their data from Google Drive.

---

## Datasets

The data is **not included** in this repository. The notebooks expect it under `MyDrive/CyberSecurity_Dataset/` on Google Drive.

### 1. DoH detection – `BCCC-CIRA-CIC-DoHBrw-2020.csv`
- **499,106 rows × 29 columns** (28 numeric features + `Label`)
- **Perfectly balanced:** 249,553 Benign / 249,553 Malicious
- Features describe each flow's statistical behaviour:
  - Flow volume and rate: `FlowBytesSent`, `FlowSentRate`, `FlowBytesReceived`, `FlowReceivedRate`
  - Packet length statistics: variance, std, mean, median, mode, skew, coefficient of variation
  - Packet time statistics: same set of measures
  - Response time statistics: same set of measures

### 2. Intrusion detection – `CyberSecurity_Dataset/Random Forest/`
- 10 attack-day CSVs, for example:
  - DoS: SlowHTTP, GoldenEye, Slowloris
  - Brute force: Web, XSS
  - SQL injection
  - Infiltration
- **571,524 rows × 323 columns** after merging, **567,440 × 317** after cleaning
- Raw columns include flow ID, timestamp, IPs and ports, plus hundreds of payload, packet-timing and inter-arrival-time statistics

---

## Pipelines

### DoH detection (`detectionmodel.ipynb`)
1. Load and merge the CSVs from the dataset folder; check nulls, duplicates (7,115 found) and class balance.
2. Visual EDA: distributions, boxplots, correlation heatmap, pair plots.
3. Cap `FlowBytesSent` outliers using the IQR rule (bounds −6,768 / 12,928).
4. Encode the label (`Benign → 0`, `Malicious → 1`).
5. Drop highly correlated features (|r| > 0.95): `FlowReceivedRate`, `PacketTimeMedian`, `ResponseTimeTimeMedian`.
6. Rank features with Random Forest importance. The top feature is `PacketLengthMode` (~0.23), followed by packet-length std/mean/median and `FlowBytesReceived`.
7. Stratified 80/20 split (399,284 train / 99,822 test, `random_state=42`).
8. Scale with `RobustScaler` and train `RandomForestClassifier(n_estimators=200, random_state=42)`.

### Intrusion detection (`Untitled36.ipynb`)
1. Concatenate the attack CSVs.
2. Drop identifiers: `flow_id`, `timestamp`, `src_ip`, `src_port`, `dst_ip`, `dst_port`.
3. Replace ±inf with NaN and fill numeric NaNs with the column median.
4. Label-encode `protocol` and `label`; drop duplicates (571,524 → 567,440 rows).
5. Winsorise every numeric column at the 1st/99th percentile.
6. Coerce to `float32`, fill remaining NaNs with 0 (316 features).
7. Stratified 80/20 split (453,952 train / 113,488 test, `random_state=42`) saved as `X_train.csv`, `X_test.csv`, `y_train.csv`, `y_test.csv`.
8. Started but not finished: engineered features (`bytes_per_packet`, `fwd_bwd_payload_ratio`, `avg_packet_gap`) and variance-threshold feature selection.

---

## Getting started

1. Open any notebook in Colab via the badge at the top, or clone the repo:
   ```bash
   git clone https://github.com/AlokChaudhary752/Trained-Model.git
   ```
2. Put the datasets in `MyDrive/CyberSecurity_Dataset/` (and the attack CSVs in `CyberSecurity_Dataset/Random Forest/`).
3. Run the notebook cells top to bottom. The first cell mounts Google Drive and installs `catboost`.

**Main libraries:** pandas, NumPy, SciPy, scikit-learn, XGBoost, CatBoost, matplotlib, seaborn.

To run locally instead of Colab, remove the `google.colab` / `drive.mount` cells and point `folder_path` at your local data directory.

---

## Results

No evaluation metrics are recorded in the notebooks yet. Accuracy, precision, recall, F1 and confusion matrices are imported in places but never computed, and no trained model file is saved. Add your numbers here once you have run the evaluation:

| Task | Model | Accuracy | Precision | Recall | F1 |
|---|---|---|---|---|---|
| DoH Benign vs. Malicious | Random Forest (200 trees) | – | – | – | – |
| Multi-class intrusion detection | Random Forest (100 trees) | – | – | – | – |

---

## Known issues to fix

- **Mixed label formats (intrusion data).** The merged `label` column contains both names (e.g. `DoS_SlowHTTP`) and numeric codes (e.g. `17`) for the same attack. `LabelEncoder` then treats them as different classes. Normalise the labels before encoding.
- **Leakage risk.** Median imputation and percentile clipping are computed on the full dataset before the train/test split. Fit them on the training set only.
- **Row removal in `Untitled32.ipynb`.** Rows with negative skew are dropped, which removes valid traffic and may bias the classes.
- **Duplicates kept in the DoH data.** 7,115 duplicate rows are detected but not removed.
- **One-hot encoding a numeric column.** `Untitled28.ipynb` one-hot encodes `FlowBytesSent`, which is a continuous feature.
- **Transforms not in the saved splits.** The log transform and engineered features in `Untitled36.ipynb` are not part of the exported train/test CSVs, because the data is reloaded from the cleaned file first.
- **Notebook names.** `UntitledNN.ipynb` names say nothing about content; consider renaming them.

---

## Roadmap

- [ ] Fix label normalisation and move preprocessing into a leakage-safe `Pipeline`
- [ ] Train and evaluate Random Forest on both tasks (accuracy, precision, recall, F1, confusion matrix, per-class report)
- [ ] Compare against Logistic Regression, Decision Tree, XGBoost, CatBoost, KNN and a voting ensemble (already imported in the notebooks)
- [ ] Hyperparameter tuning with `GridSearchCV` and cross-validation
- [ ] Finish feature selection and feature engineering for the intrusion data
- [ ] Save the final models (e.g. with `pickle`/`joblib`) and add an inference example
- [ ] Add a `requirements.txt` and a short data-download guide

---

## Author

**Alok Chaudhary** – [@AlokChaudhary752](https://github.com/AlokChaudhary752)
