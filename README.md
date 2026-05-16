# Resort Guest Churn Prediction

This notebook walks through the full machine learning pipeline we built to predict which resort guests are likely to cancel their stay (churn). It covers everything from loading the data to generating a Kaggle submission file.

The competition task: given information about a guest's booking, demographics, and spending habits, predict whether they will churn (1) or stay (0).

---

## What's in the notebook

1. **Load the data** — read the train and test CSVs
2. **Exploratory analysis** — charts to understand the data and key patterns
3. **Data cleaning** — fix known issues in the raw data
4. **Feature engineering** — create new columns that improve model performance
5. **Train the model** — CatBoost with 5-fold cross-validation
6. **Hyperparameter tuning** — Optuna to find the best settings
7. **Evaluate** — confusion matrix, ROC curve, feature importances
8. **Make predictions** — run the model on the test set and save a submission file

---

## How to run it

**1. Create a virtual environment (optional but recommended)**

```bash
python -m venv venv
venv\Scripts\activate      # Windows
source venv/bin/activate   # Mac / Linux
```

**2. Install dependencies**

```bash
pip install -r requirements.txt
```

**3. Open the notebook**

```bash
jupyter notebook resort_pipeline.ipynb
```

Then run all cells top to bottom. The Optuna tuning step (section 7) takes a few minutes — that's normal.

---

## Files

```
resort v1/
├── data/
│   ├── resort_train.csv      training data (6,954 guests)
│   └── resort_test.csv       test data to predict on (1,739 guests)
├── resort_pipeline.ipynb     the main notebook
├── requirements.txt          Python packages needed
└── README.md                 this file
```

A `submission.csv` file will be created in the root folder when you run section 9.
