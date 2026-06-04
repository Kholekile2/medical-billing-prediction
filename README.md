# Predictive Analytics for Medical Scheme Claims Processing

Predicting processing delays in private patient medical billing using machine learning.

This project was completed as part of the BIA 716 Applied Research module at the University of the Western Cape. It builds and compares four classification models to predict whether a private patient invoice will be processed on time or delayed beyond 14 days, using only information available at the point of invoice capture.

## Project Overview

Medical billing bureaus face cash flow pressure and bad debt risk when invoices take too long to process. This study develops a predictive model that flags high-risk invoices at the point of capture so the bureau can act early rather than chasing delays after they happen.

The final model is an XGBoost classifier that achieved a ROC-AUC of 0.7570 and recall of 70.04% on a held-out test set of 15,000 records.

## Dataset

- **Source:** Private patient billing records (self-pay, no medical scheme)
- **Size:** 50,000 invoices
- **Target:** Processing Class — Timely (75.9%) or Delayed (24.1%), based on a 14-day processing threshold
- **Predictors (6):** Posted Billing Group, Specialty, Debtor Status, Age Bracket, ICD10 Chapter, Facility Type

The raw dataset is not included in this repository for confidentiality reasons.

## Repository Structure

```
MedicalBillingResearch/
├── dataset/
│   ├── PRIVATE PATIENT.xlsx          (raw data - not committed)
│   ├── private_patient_clean.csv     (after cleaning)
│   ├── private_patient_final.csv     (after feature engineering)
│   ├── train_set.csv                 (70% training split)
│   └── test_set.csv                  (30% test split)
├── notebooks/
│   ├── 01_data_understanding.ipynb
│   ├── 02_data_preparation.ipynb
│   └── 03_modelling/
│       ├── 03a_logistic_regression.ipynb
│       ├── 03b_decision_tree.ipynb
│       ├── 03c_random_forest.ipynb
│       ├── 03d_model_comparison.ipynb
│       ├── 03e_shap_analysis.ipynb
│       └── 03f_xgboost.ipynb
├── outputs/
│   └── figures/                      (all generated plots)
├── models/                           (saved metrics, predictions, importance)
├── README.md
└── requirements.txt
```

## How to Reproduce

### 1. Set up the environment

```bash
python -m venv .venv
.venv\Scripts\activate          # Windows
pip install -r requirements.txt
```

### 2. Run the notebooks in order

1. `01_data_understanding.ipynb` — explores the raw data
2. `02_data_preparation.ipynb` — cleans data, engineers features, creates train/test split
3. `03a` through `03c` — builds Logistic Regression, Decision Tree, Random Forest
4. `03f_xgboost.ipynb` — builds XGBoost
5. `03d_model_comparison.ipynb` — compares all four models, runs bootstrap analysis
6. `03e_shap_analysis.ipynb` — SHAP explainability on the selected model

All notebooks use `random_state=42` for reproducibility.

## Methodology Summary

1. **Data preparation** — reduced from 32 raw variables to 6 predictors through quality screening, association testing (Cramer's V), and multicollinearity assessment. No records were dropped.
2. **Modelling** — four classifiers built with class imbalance handling (balanced class weights / scale_pos_weight).
3. **Model selection** — bootstrap confidence intervals (1,000 iterations) used to compare Random Forest and XGBoost statistically.
4. **Explainability** — SHAP analysis on the selected XGBoost model for global importance, direction of effect, interactions, and individual predictions.

## Key Findings

- **Facility Type** and **Specialty** are the strongest predictors of delay.
- **Hospital** invoices and **Mental Health / General Practice** specialties are the most likely to be delayed.
- **Consulting Rooms** and **Dental / Obstetrics / Ophthalmology** specialties are the most likely to be timely.
- **Age Bracket** strongly predicts timely processing for young invoices.
- **Debtor Status** contributes the least once clinical and facility characteristics are known.

## Model Performance (Test Set, n = 15,000)

| Model | Accuracy | ROC-AUC | F1 (Delayed) | Recall |
|-------|----------|---------|--------------|--------|
| Logistic Regression | 63.99% | 0.6973 | 0.4739 | 67.38% |
| Decision Tree | 67.05% | 0.7251 | 0.4888 | 65.44% |
| Random Forest | 68.62% | 0.7550 | 0.5144 | 69.04% |
| **XGBoost (selected)** | 68.15% | **0.7570** | 0.5143 | **70.04%** |

## Technologies

- Python 3.12
- pandas, numpy — data handling
- scikit-learn — Logistic Regression, Decision Tree, Random Forest, metrics
- xgboost — gradient boosting model
- shap — model explainability
- matplotlib, seaborn — visualisation

## Author

Kholekile Mpengesi
University of the Western Cape — BIA 716 Applied Research Project
Supervisor: Ms Chanel Morkel

## License

This project is submitted for academic assessment. The dataset is confidential and not included in this repository.