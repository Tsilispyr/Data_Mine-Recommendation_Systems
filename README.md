# Data_Mine-Recommendation_Systems

# SFO Air Traffic Classification - Data Mining and Recommender Systems

## Project: Εξόρυξη Δεδομένων και Συστήματα Συστάσεων (Data Mining and Recommender Systems), 2025-2026
## Title: Πρόβλεψη Επιβατικής Κίνησης, 5-Class Classification ("Forecasting Airport Passenger Traffic Volume")


## Contents

| File | Role |
|---|---|
| [`data mine recom syst.knwf`](<data mine recom syst.knwf>) | The KNIME Analytics Platform workflow export, roughly 120 nodes. This is the entire codebase. |
| [`Εξόρυξη Δεδομένων και Συστήματα Συστάσεων.pdf`](<Εξόρυξη Δεδομένων και Συστήματα Συστάσεων.pdf>) | 30-page written report: methodology, per-model result graphs and analysis, final comparison, business conclusions. |
| [`Presentation Εξόρυξη Δεδομένων και Συστήματα Συστάσεων.pptx`](<Presentation Εξόρυξη Δεδομένων και Συστήματα Συστάσεων.pptx>) | 15-slide summary presentation of the same material. |

## Dataset

Two San Francisco International Airport open datasets (Passenger statistics and Landing statistics, 2000-2022) are joined into one table. The target is passenger count, binned into 5 classes A through E (roughly: Class A at 30,000+ passengers down to Class E under 3,500), turning what would naturally be a regression problem into the 5-class classification the assignment required.

## Data preparation pipeline

1. **Data integration**: two CSV Reader nodes plus a Joiner combine the Passengers and Landings files, enriching each passenger record with landing-based features (landing count, landed weight, aircraft/airline details).
2. **Data cleaning**:
   - Duplicate Row Filter removes repeated records, which would otherwise bias the class statistics.
   - Missing Value imputes numeric columns with the median and categorical columns with the most frequent value; median was chosen over mean for robustness to outliers.
   - Numeric Outliers applies IQR-based bounds to every numeric column except Activity Period (a date field with no meaningful "outlier" in the statistical sense). This removed 5,791 rows over the Passenger Count upper bound (about 42,502.5), 4,616 rows over the Landing Count upper bound (188 landings), and 3,779 rows over the Total Landed Weight upper bound (about 44 million lbs).
3. **Feature engineering**: a Numeric Binner turns Passenger Count into the 5 target classes (A-E); a Column Filter then drops the raw Passenger Count column to avoid data leakage, since leaving the numeric answer in the input would let a model read it directly and score 100% without learning anything.
4. **Partitioning**: a Domain Calculator recomputes valid column domains after filtering, so downstream nodes do not expect classes that no longer exist. A Row Sampler performs a 70/30 stratified split by class, preserving each class's original proportion in both sets. A Reference Row Filter then derives the test set as the exact complement of the training set by RowID, rather than relying on a single partitioning node, to guarantee the two sets never overlap.

## Modeling approach

Every classifier is wrapped in the same training-versus-test comparison pattern to catch overfitting: an Interval Loop Start or Parameter Optimization Loop Start sweeps one hyperparameter, a Learner/Predictor pair scores both the training set and the held-out test set at each iteration, Variable to Table Row/Column nodes plus a Joiner assemble a results table, and a Line Plot with a Sorter chart training accuracy against test accuracy across the swept values to locate both the point of divergence (overfitting) and the best test-set score.

Models tried, with the parameter swept in each loop:

- **Decision Tree**: Min Records per Node (2, 4, 6, ...)
- **Gradient Boosted Trees**: Min Node Size and Number of Models
- **Naive Bayes**: Minimum Standard Deviation
- **Random Forest**: Min Records per Node / Tree Depth
- **k-Nearest Neighbors**: k (3 to 21 in steps of 2), after One-Hot Encoding the categorical columns (One to Many) and Min-Max normalizing all features (a Normalizer fit on the training set only, then applied to the test set, to avoid data leakage; the report also notes the resulting "out of bounds" values, on the order of 1e-5, where the test set's min/max fall just outside the training set's [0,1] range)
- **SVM**: the Cost (C) parameter, with the same normalization and one-hot encoding as KNN
- **RProp MLP**: number of training epochs

## Results

Native KNIME models, ranked by test accuracy:

| Rank | Model | Test accuracy | Training accuracy | Note |
|---|---|---|---|---|
| 1 | Random Forest | 74.8% | 94.6% | Best generalization despite a roughly 20-point train/test gap; chosen as the final model. |
| 2 | Gradient Boosted Trees | 70.9% | 100% | Clear overfitting; needed min node size = 10 to reach this from a much worse default. |
| 3 | Decision Tree | 66.0% | 67.1% | Smallest train/test gap (1.1 points) of any model; an honest, well-balanced baseline. |
| 4 | RProp MLP | 62.6% | 85.7% | Large overfitting gap (23 points); a neural net needs more data than this tabular set provides. |
| 5 | KNN | 51.9% (k=21) | 59.9% | Curse of dimensionality after one-hot encoding many categorical columns. |
| 6 | Naive Bayes | 51.4% | 54.5% | Underfits; the feature-independence assumption is too strong for this data. |

A further optimization pass was done outside the native KNIME learner nodes, in two embedded Python (scikit-learn) scripting nodes inside the workflow:

- A small grid search over Random Forest's `min_samples_leaf` (1 or 2) and `max_features` (0.4, 0.5, or 0.6) at `n_estimators=800` raised test accuracy to **78.6%** (`min_samples_leaf=2`, `max_features=0.5` won).
- A soft-voting ensemble combining that tuned Random Forest (`n_estimators=500`, `min_samples_leaf=2`, `max_features=0.5`) with a tuned Gradient Boosting Classifier (`n_estimators=500`, `learning_rate=0.1`, `max_depth=8`, `subsample=0.8`) reached **78.9%**, essentially level with Random Forest alone; the report treats the 0.3-point gain as noise rather than a real improvement.
- The winning Random Forest's feature importances rank Activity Period (time of year, over 25% importance, reflecting seasonality), airline identity (carrier and IATA code), and GEO Region (US versus international, 7.2%) as the strongest predictors.

## Conclusions

- Tree-based ensembles win outright on this tabular problem. Random Forest is the recommended final model, at 74.8% test accuracy natively in KNIME and 78.6% to 78.9% after the scikit-learn tuning and ensembling pass.
- Distance-based and probabilistic models (KNN and Naive Bayes) both land near 51 to 52%, only slightly better than a naive majority-class guess, due to the curse of dimensionality from one-hot encoding and the independence assumption respectively.
- The RProp MLP overfits and underperforms Random Forest despite far more parameters, matching the general finding that neural networks need much larger datasets to beat tree ensembles on structured, tabular data.
- Business framing (from the report and presentation): correctly predicting the passenger-traffic class 3 times out of 4 lets airport operations plan staffing, such as security and cleaning, ahead of time instead of relying on ad hoc estimates.

## How to open

Import `data mine recom syst.knwf` into KNIME Analytics Platform (File > Import KNIME Workflow); the workflow metadata records KNIME version 5.8.1. The two Python scripting nodes used for the final optimization pass require KNIME's Python integration configured with `pandas`, `scikit-learn`, `matplotlib`, and `numpy`. The raw SFO open-data CSVs (Passengers and Landings) are not included in this folder, so the workflow's CSV Reader nodes will need to be repointed at local copies before it can be re-executed.
