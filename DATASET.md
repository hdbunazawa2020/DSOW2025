# Data Science Osaka Winter 2025 – Dataset Specification

## Dataset Overview

| Item | Value |
|------|-------|
| **Number of Files** | 4 (train.csv, test.csv, supplemental.csv, sample_submission.csv) |
| **Training Samples** | (see `train.csv`) |
| **Test Samples** | (see `test.csv`) |
| **Primary Input** | SMILES (chemical structure representation) |
| **Target Variables** | Classification: Thermotropic LC (0/1) ; Regression: Clearing Point (°C) |
| **Submission Target** | One string field: `"[probability, temperature]"` per `row_id` |

> Note: The exact row counts and full column list can be confirmed by opening each CSV, as Kaggle datasets sometimes evolve.

---

## File Structure

### train.csv (Training Dataset)

学習データ。分子の情報（SMILES等）に加え、液晶性ラベル・透明点などが含まれます。モデルの学習・CVに使用します。

- **Contains**: identifiers + features + targets
- **Used for**: training / validation / feature analysis

### test.csv (Test Dataset)

推論対象データ。`train.csv` と同様の入力カラムを持ちますが、目的変数は含まれません。

- **Contains**: identifiers + features
- **Used for**: generating submission predictions

> Competition note: `test.csv` に含まれる Thermotropic LC は必ず透明点を持つ（熱分解やモノトロピックは含まれない）と説明されています。

### supplemental.csv (Supplemental / Reference Dataset)

参考データ。Thermotropic LC 性が示唆される一方で、等方相転移前に熱分解する等の理由で **透明点（Clearing Point）が欠損**しているサンプルを含みます。

- **Good for**: classification の追加学習・半教師/事前学習・特徴分布の把握
- **Not directly usable for**: regression の教師データ（透明点が欠損）

### sample_submission.csv (Sample Submission)

提出フォーマット確認用ファイル。**これをベースに prediction を埋める**と事故りにくいです。

例（フォーマット）：

```csv
row_id,prediction
891,"[0.05, 0.0]"
892,"[0.88, 145.5]"
893,"[0.45, 120.0]"
...
```

---

## Columns (Conceptual)

Kaggleページの説明から推定できる「必須概念カラム」は以下です。  
実際のカラム名は `train.csv` / `test.csv` を開いて確認してください（環境により表記が変わることがあります）。

### Identifier

| Column | Data Type | Description |
|--------|-----------|-------------|
| **row_id** | Integer | Unique row identifier used for submission join |

### Primary Feature

| Column | Data Type | Description |
|--------|-----------|-------------|
| **smiles** | String | SMILES representation of the molecule (chemical structure) |

> 多くの場合、`smiles` 以外に分子に関する付加情報（実験条件に近いメタデータ、構造の別表現、出典など）が含まれる可能性があります。

### Targets (train.csv only)

| Column | Data Type | Values / Range | Description |
|--------|-----------|----------------|-------------|
| **is_thermotropic_lc** | Integer | 0, 1 | 1 = Thermotropic LC, 0 = non-LC |
| **clearing_point_celsius** | Float | (°C) | Clearing point (LC → isotropic transition temperature) |

> 注意:
> - 回帰の評価は「真のラベルが 1（LC）」のみで計算されます
> - `supplemental.csv` は clearing point が欠損していることがあります

---

## Data Quality / Real-World Noise Considerations

このデータは現実の材料データ由来であり、説明上も以下の曖昧さが含まれます。

- 相転移温度の測定手法（POM / DSC）が区別されていない
- 昇温過程 / 降温過程が明確とは限らない
- したがって **ある程度のラベルノイズ / 測定ばらつき**がある前提で頑健に扱うのが望ましい

---

## Feature Engineering Opportunities (Chem-informatics)

### 1) RDKit Fingerprints (Strong Baseline)
- Morgan / ECFP (radius=2, nBits=2048 など)
- count-based fingerprint も試す価値あり（木モデルと相性が良いことがある）

### 2) RDKit Descriptors
例:
- MolWt, LogP, TPSA, NumHDonors, NumHAcceptors
- NumRotatableBonds, RingCount, Aromatic proportion, FractionCSP3

> NaN / inf が出ることがあるので、クリーニング & 欠損処理を用意する。

### 3) SMILES Text / Embeddings
- ChemBERTa 等の SMILES Transformer で embedding を作り、木モデル or 小さなNN headに入力
- canonical smiles を揃える（RDKitで正規化）と安定しやすい

---

## Handling supplemental.csv

`supplemental.csv` は「透明点が欠損し得る」ため、代表的な使い方は以下です。

- **分類の追加教師データとして利用**（ラベルが明確に 1 と扱える場合）
- **事前学習 / 自己教師あり**（SMILESの言語モデル、対比学習）
- **回帰には使わない**（clearing point がないため）

> 使い方の可否は competition rules に従うこと。

---

## Submission Format (Critical)

### Required Columns

| Column | Data Type | Description |
|--------|-----------|-------------|
| `row_id` | Integer | Must match `test.csv` row_id |
| `prediction` | String | Must be a string like `"[probability, temperature]"` |

### Prediction Rules
- `probability`: float in [0, 1]
- `temperature`: float (°C)
- 非液晶と予測する場合でも温度を入れる（例: `0.0`）

### Example Submission

```python
import pandas as pd

sub = pd.read_csv("sample_submission.csv")

# example: two numpy arrays or lists
# prob_pred: shape (n_test,)
# temp_pred: shape (n_test,)

sub["prediction"] = [
    f"[{p:.6f}, {t:.6f}]"
    for p, t in zip(prob_pred, temp_pred)
]

sub.to_csv("submission.csv", index=False)
```

---

## Data Loading Example

```python
import pandas as pd

train = pd.read_csv("train.csv")
test = pd.read_csv("test.csv")
supp = pd.read_csv("supplemental.csv")

print(train.shape, test.shape, supp.shape)
print(train.head())
print(train.isnull().sum().sort_values(ascending=False).head(20))
```

---

## Reference Links

- Kaggle Competition Data Page: https://kaggle.com/competitions/data-science-osaka-winter-2025/data
- Competition Overview: https://kaggle.com/competitions/data-science-osaka-winter-2025
