# Forecasting Train Occupancy on the Paris Transilien Network

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/efeleau/transilien-occupancy-forecasting/blob/main/SNCF.ipynb)

This project predicts **how full a train will be at its next station**. It is based on the [SNCF-Transilien Data Challenge](https://challengedata.ens.fr/challenges/89) (Collège de France / ENS), with real passenger counts from infrared door sensors on Paris suburban trains.

The model beats SNCF's official benchmark by 15% and has an average error of **1.1 occupancy points on the hidden test set**.

*The notebook is written and commented in French.*

## The problem

For each stop, the task is to predict the **occupancy rate** (passengers on board divided by train capacity, between 0 and 1) using what is already known at prediction time:

| Variable | Meaning |
|---|---|
| `train`, `station` | Train ID (1 to 55) and station ID (36 stations) |
| `date`, `hour` | Day and time slot |
| `p0q1`, `p0q2`, `p0q3` | Occupancy of the **same train** at the 1, 2 and 3 previous stations |
| `p1q0`, `p2q0`, `p3q0` | Occupancy of the 1, 2 and 3 **previous trains** at the same station |

Data: 31,119 stops for training (January to May 2019) and 13,752 stops for testing (late May to early July 2019).

## Results

Validation set: the last 14 days of the training period (15 April to 20 May), never used for training.

| Model | MAE | RMSE |
|---|---|---|
| Global mean | 0.1260 | 0.150 |
| Mean per (train, station) pair | 0.0267 | 0.039 |
| Naive: "same as the previous station" | 0.0638 | 0.088 |
| SNCF official benchmark (linear regression per pair) | 0.0121 | 0.0227 |
| **Gradient boosting, 5 features (final model)** | **0.0103** | **0.0186** |

**Hidden test set (official challenge score): MAE = 0.0112**, i.e. an average error of 1.1 occupancy points on weeks the model never saw. This is only 9% above the validation MAE, which confirms that the time-based validation gave a reliable estimate of real-world performance.

## Key findings

1. **The (train, station) pair carries most of the information.** Knowing only which train stops at which station explains 94% of the variance in occupancy: a given train is filled about the same way at a given station every day.
2. **Day-to-day variations come from the same train upstream.** If the train is fuller than usual at the previous stations, it will be fuller at the next one. The final model only needs 5 features: `train`, `station`, `p0q1`, `p0q2`, `p0q3`. Previous trains, the time slot and the day of the week add nothing.
3. **Correlated is not redundant.** `p0q2` and `p0q3` are 0.91 correlated with `p0q1`, yet removing them increases the error by 25%. Several measurements of the same train probably help the model correct sensor errors.
4. **Features matter far more than hyperparameters.** The best of 12 hyperparameter settings improves the model by only 1.6%, a gap within noise.
5. **High correlation does not mean a good forecast.** `p0q1` is the feature most correlated with the target, yet predicting "same as the previous station" is twice as bad as the per-pair mean: trains fill up on their way to Paris, so this forecast is systematically too low.

## Approach

1. **Exploratory analysis**, which drove every later decision:
   - the test set covers dates *after* the training set, with no overlap;
   - only regular weekdays are present (school holidays and bank holidays are excluded);
   - missing values carry meaning: a missing `p0q1` or `hour` marks a train's first station (almost empty, 7.7% average occupancy), so they were kept as missing rather than imputed;
   - `way` and `composition` are constant and were dropped; `month` was dropped because June and July never appear in training.
2. **Time-based validation** (train up to 12 April, validate from 15 April) to reproduce the test situation. A random K-fold would mix stops from the same day between training and validation, and overestimate performance.
3. **Baselines**, from the global mean to SNCF's official benchmark, to set the score to beat.
4. **Gradient boosting** (`HistGradientBoostingRegressor`), chosen for its native handling of missing values and categorical features. In a quick comparison it matched a random forest in MAE (0.0108) while training about 9 times faster, and beat a Ridge regression (0.0117).
5. **Hyperparameter grid search and ablation study** to measure what really drives performance, leading to a simpler 5-feature model.
6. **Final model** retrained on all training data, then submitted to the challenge.

## Limitations

- **An unseen test period.** The test set covers late May to early July, a period absent from training: a seasonal effect not reflected in the lag features could not be anticipated.
- **A single validation window** of 14 days, some of them atypical (right after the spring holidays, May bank-holiday weekends).
- **Imperfect sensor data.** Some days are incomplete, and measurement errors are likely.
- **Smoothed predictions.** The model predicts the most likely occupancy and cannot anticipate exceptional situations such as incidents.

## How to run

**Data.** The data files (`Xtrain.csv`, `Ytrain.csv`, `Xtest.csv`) are not included in this repository. Download them from the [challenge page](https://challengedata.ens.fr/challenges/89).

**Colab (recommended).** Click the badge above and run all cells. The notebook asks you to upload the three CSV files. It runs in about two minutes.

## Background

This project started as a course project in the Master's in Statistics & Machine Learning at Sorbonne Université.

