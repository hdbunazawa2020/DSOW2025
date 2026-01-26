# Data Science Osaka Winter 2025 – Exploring the World of Chemical Data

## Competition Overview

| Item | Value |
|------|-------|
| **Competition** | Data Science Osaka Winter 2025 – Exploring the World of Chemical Data |
| **Task Type** | **Multi-task**: Binary Classification + Regression |
| **Problem Class** | Supervised Learning (Cheminformatics) |
| **Metric** | **Decoupled Rank Score** = 0.5 × Normalized Gini + 0.5 × Spearman’s ρ |
| **Optimization Direction** | **Maximize** (higher is better; typically -1.0 to +1.0) |
| **Competition Type** | Limited-time Kaggle Competition |
| **URL** | https://kaggle.com/competitions/data-science-osaka-winter-2025 |

---

## Background

本コンペは、有機分子の**相転移挙動**を予測して材料探索を加速することを目的としています。  
サーモトロピック液晶（Thermotropic LC）相を示す分子は、LCD、光学デバイス、センサー等において重要な基礎材料ですが、新規材料探索は合成と評価を繰り返す試行錯誤になりがちです。

機械学習で以下を推定できれば、
- **不要な実験（空振り）**を減らし
- **有望材料の見逃し**を減らしつつ
- 合成すべき候補を優先順位付け

できるため、探索効率の向上が期待されます。

---

## Competition Goal

各分子に対して、以下 2 つを同時に予測します。

1) **Classification**: その分子が Thermotropic LC である確率（0〜1）  
2) **Regression**: 液晶候補分子の **透明点 (Clearing Point, °C)** を予測

---

## Task Definition

### Inputs
- 主に **SMILES（化学構造）**
- その他 `train.csv` に含まれる追加カラム（提供されている場合）

### Outputs
各 `row_id` について、2つの値を含むリストの**文字列表現**を出力します。

- `probability`: Thermotropic LC である確率（float, 0.0〜1.0）
- `temperature`: 透明点（°C, float）

---

## Key Constraints and Notes

### Data Notes
- `test.csv` に含まれる **Thermotropic LC は必ず透明点を持つ**  
  （モノトロピックや「分解して透明点が取れない」ケースは含まれない想定）
- `supplemental.csv` は参考データ：
  - Thermotropic LC 性が示唆されるが、等方相転移前に熱分解する等の理由で**透明点が欠損**しているものを含む
  - 使う場合は主に **分類タスクの補強**としての扱いが自然
- 相転移温度等は **POM / DSC の区別なし**
- **昇温 / 降温**の区別も明確とは限らない  
  → 現実データらしい「ラフさ」があり、頑健性が重要

### Submission Notes
- 非液晶と予測する場合（確率が低い場合）でも、**提出フォーマット上は温度を必ず含める**（例: `0.0`）
- ただし回帰評価は「真のラベルが 1（液晶）」のサンプルだけで計算されるため、
  - 真の非液晶（label=0）の温度予測はスコアに影響しない  
  - しかし **test の真のラベルは不明**なので、推論では温度もちゃんと出すのが無難

---

## Evaluation Metric (Decoupled Rank Score)

最終スコアは、次の 2 指標の **等重み平均（0.5ずつ）**です。

### Part A: Classification (Normalized Gini)
全データを対象に ROC AUC を計算し、次で正規化ジニに変換します。

- **NormGini = 2 × AUC − 1**
- 範囲: -1.0 〜 +1.0（概念上）
- 解釈: ランダム予測 ≈ 0、完全に正しく順位付けできると 1（完全逆順だと -1）

### Part B: Regression (Spearman’s rho)
真のラベルが **1（液晶）** のデータのみを対象に、真の透明点と予測温度の **スピアマン順位相関**を計算します。

- 絶対誤差（バイアス）は無視され、**順位（大小関係）**が合っているかを評価
- 真の非液晶（label=0）は回帰評価から除外

### Final
- **FinalScore = 0.5 × NormGini + 0.5 × Spearman ρ**

---

## Submission Format

提出 CSV は以下の形式：

- columns: `row_id`, `prediction`
- `prediction` は **文字列**（ダブルクォート推奨）

例：

```csv
row_id,prediction
891,"[0.05, 0.0]"
892,"[0.88, 145.5]"
893,"[0.45, 120.0]"
```

---

## Dataset Files

- `train.csv`  
  学習用データ。液晶性（ラベル）と透明点などを含む。
- `test.csv`  
  推論対象データ。ラベル・透明点は含まれない。
- `supplemental.csv`  
  参考データ。液晶性が示唆されるが透明点が欠損しているデータを含む（必要に応じて活用）。
- `sample_submission.csv`  
  提出フォーマット確認用。**このファイルをベースに prediction を埋める**のが安全。

---

## Recommended Approach (Practical)

### 1) Baseline（まず動く最小構成）
- RDKitで
  - Morgan Fingerprint（ECFP）
  - 主要 descriptors（MolWt, LogP, TPSA, HBD/HBA, Rings など）
- モデル
  - 分類：LightGBM / CatBoost（AUC最適化）
  - 回帰：LightGBM / CatBoost（Spearman を fold 内で計測、正例のみ）

### 2) CV 設計（メトリクスに忠実に）
- 分類：Stratified KFold（label で層化）
- 回帰：fold 内で「正例のみ」で Spearman を計測
- ログ：
  - AUC → NormGini
  - Spearman（positives only）
  - FinalScore（0.5/0.5）

### 3) アップグレード案（効く順に）
- Fingerprintの種類やサイズ（bit数、count版）
- Descriptor の追加とクリーニング（NaN/inf対策）
- テキスト表現（ChemBERTa 等の埋め込み）＋ 木モデル or 小さなhead
- アンサンブル（平均・rank平均）  
  ※順位系の指標はアンサンブルが効きやすいことが多い

---

## Timeline (From Competition Page)

- 12/22 18:00–19:00: コンペ開始・ルール説明
- 1/19 18:00–19:00: 中間発表・スコア向上ヒント
- 2/9 9:00: コンペ終了
- 2/9 18:00–19:00: 最終発表・上位者表彰・ソリューション発表

---

## Citation

YuyaYamamoto. Data Science Osaka Winter 2025. https://kaggle.com/competitions/data-science-osaka-winter-2025, 2025. Kaggle.
