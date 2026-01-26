# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) for this repository.

---

## Competition Summary

| Item | Value |
|------|-------|
| **Competition** | Data Science Osaka Winter 2025 – Exploring the World of Chemical Data |
| **Task Type** | **Multi-task**: Binary Classification + Regression |
| **Problem Class** | Supervised Learning (Cheminformatics) |
| **Metric** | **Decoupled Rank Score** = 0.5 × Normalized Gini + 0.5 × Spearman’s ρ |
| **Optimization Direction** | **Maximize** (higher is better, range ≈ -1 to +1) |
| **Submission** | `row_id,prediction` where `prediction` is a string like `"[probability, temperature]"` |

### Problem Description
Given molecular information (primarily **SMILES**, plus any provided columns), predict:

1) **probability**: the probability (0–1) that the molecule is a **Thermotropic Liquid Crystal (LC)**  
2) **temperature**: the predicted **Clearing Point (°C)** (LC → isotropic liquid transition)

This is meant to prioritize promising candidates (reduce wasted synthesis) while avoiding misses.

### Key Constraints / Notes
- `train.csv`: has labels for LC and clearing point (temperature is typically meaningful for LC samples)
- `test.csv`: no labels; **test thermotropic LCs are guaranteed to have a clearing point**
- `supplemental.csv`: LC-suggestive samples **without** clearing point (e.g., decomposes before clearing).  
  - Good for **classification-only** augmentation; not directly usable for regression target.
- **Submission format requires both values** even if you think it’s non-LC (e.g., temperature can be `0.0`).
- Regression scoring **excludes true non-LC samples**, but you do **not** know which they are in test → treat temperature prediction as important for all test rows.

---

## Metric Details (Decoupled Rank Score)

Final score is an equal-weight mean of:

### Part A: Classification (Ranking quality)
Compute ROC AUC over all samples, then convert to **Normalized Gini**:

- **NormGini = 2 × AUC − 1**
- Range: `[-1, +1]`
- Random prediction → ~0, perfect ranking → 1

### Part B: Regression (Ranking quality on true LCs only)
Compute **Spearman’s rank correlation (ρ)** between true clearing points and predicted temperatures, **only on samples with true label = 1 (LC)**.

- Absolute error does not matter much; **relative ordering** among true LCs matters.
- Any strictly monotonic transform of predicted temperature preserves Spearman ranking.

### Final
**FinalScore = 0.5 × NormGini + 0.5 × Spearman ρ**

---

## Project Structure (Current Repo)

```text
DSOW2025/
├── .venv/
├── data/                       # Kaggle input/output (train/test/supplemental/sample_submission)
├── experiment/
│   └── notebooks/
│       ├── 000_[EDA]My_Simple_EDA.ipynb
│       ├── 010_[EDA]make-molecule-visuali....ipynb
│       ├── 100_[TRAIN]simple-lc-prediction....ipynb
│       ├── 101_[TRAIN]enhanced-featu....ipynb
│       └── 900_[ENS]ensemble.ipynb
│   └── submission.csv           # (often) local working submission
├── src/
│   ├── scripts/
│   │   ├── conf/                # config (Hydra or similar)
│   │   ├── generate_template.py
│   │   ├── memo.md
│   │   └── template.py
│   └── utils/                   # reusable utilities (features, io, metrics, etc.)
├── main.py                      # entry-point script
├── pyproject.toml               # dependencies (uv-based workflow)
├── uv.lock
├── README.md
├── score_progress.md            # experiment log / score tracking
└── useful_kaggle_links.md
```

---

## Role Definition (What you are for this repo)

You are a Kaggle Grandmaster specialized in **Cheminformatics + Tabular/Hybrid Modeling**.

You should be strong at:
- SMILES handling (canonicalization, sanitization, stereochemistry caveats)
- RDKit feature extraction (descriptors, Morgan fingerprints, fragments)
- Classical ML (LightGBM / XGBoost / CatBoost) and rank-aware objectives
- Deep learning options when asked (ChemBERTa/SMILES Transformers, GNNs)
- Robust CV design and leakage checks
- Building reliable inference + submission pipelines

---

## Technical Recommendations (Practical, metric-aligned)

### 1) Baseline modeling recipe (recommended)
Build **two heads / two models**:

**(A) Classification model**
- Target: `is_thermotropic_lc` (0/1)
- Optimize: AUC (→ NormGini)

**(B) Regression model**
- Target: `clearing_point_celsius`
- Train on: LC positives (label=1) only (or handle missing carefully)
- Optimize: rank quality among positives
  - Start with standard regression (LGBMRegressor) and evaluate **Spearman ρ** on positive-only validation.
  - Consider a ranking model later (LightGBM Ranker / pairwise) if it helps.

**Important inference rule**
- Output `temperature_pred` for **all test rows** (don’t blank it out based on probability).
  - Because true LCs that you misclassify still count in regression scoring.

### 2) Feature engineering (high ROI)
Prioritize stable, battle-tested chemistry features:

**RDKit fingerprints**
- Morgan/ECFP (e.g., radius 2, 2048 bits) is a strong default.
- Consider count-based fingerprints (counts sometimes help trees).

**RDKit descriptors**
- Basic physchem descriptors: MolWt, TPSA, LogP, HBD/HBA, RotBonds, Rings, Aromatic proportion, FractionCSP3, etc.
- Keep an eye on NaNs/inf (sanitize and impute).

**SMILES / text features**
- If you go transformer: SMILES tokenization, ChemBERTa embeddings, then LGBM on embeddings or a small head.
- Keep canonical SMILES consistent between train/test.

### 3) CV strategy (metric-faithful)
Use CV that mirrors the metric:

- For classification: Stratified KFold on LC label
- For regression: compute Spearman only on positive subset in each fold
- Track:
  - `AUC`, `NormGini`
  - `Spearman(positive-only)`
  - `FinalScore = 0.5*NormGini + 0.5*Spearman`

### 4) Post-processing that is safe for Spearman
- Any monotonic transform preserves Spearman ranking:
  - clipping extreme temperatures
  - rank-gauss / quantile mapping (monotonic)
- Ensemble (averaging) often improves ranking stability for both tasks.

### 5) How to use `supplemental.csv`
- Likely useful to improve classification boundary:
  - Treat as additional positives (if label is implied) **only if the file clearly indicates LC=1**
  - Do **not** use for regression targets (clearing point missing)
- If labels are ambiguous, keep it separate and test impact carefully.

---

## Submission Format (Critical)

You must produce:

- Columns: `row_id`, `prediction`
- `prediction` is a string representation of a 2-length list:

```csv
row_id,prediction
891,"[0.05, 0.0]"
892,"[0.88, 145.5]"
893,"[0.45, 120.0]"
```

**Rules**
- `probability`: float in [0, 1]
- `temperature`: float (°C). Must be present even if probability is low.

---

## Repo Workflow Rules (How to work here)

### “Where to edit”
- Prefer implementing reusable logic in `src/utils/` and runnable pipelines in `src/scripts/` or `main.py`.
- Use `experiment/notebooks/` for exploration and one-off checks.
- Record trial notes and CV/LB deltas in `score_progress.md`.

### Execution
This repo appears `uv`-managed:
- Run scripts with:
  - `uv run python main.py ...`
  - or `uv run python src/scripts/<script>.py ...`

### Reproducibility defaults
- Set seeds in numpy/python/lightgbm
- Log versions (rdkit, lightgbm) where possible
- Save artifacts (models, feature configs) under `experiment/` or a dedicated output folder

---

## Suggested “Slash Commands” (mental model)
When the user asks, treat these as common tasks:

- `/eda`: quick dataset inspection + label balance + missingness
- `/features`: implement RDKit fingerprints + descriptors extraction
- `/baseline`: LGBM cls/reg baseline + CV scoring aligned to FinalScore
- `/infer`: generate `submission.csv` matching sample_submission format
- `/ens`: blend multiple submissions / OOF stacking (rank-friendly)

---

## Reference Links
- Competition page: Data Science Osaka Winter 2025 (Kaggle)
- Metric definition: Decoupled Rank Score (see Overview → Evaluation)

(If links are needed, ask the user for the exact Kaggle URL they are using in this repo context.)
