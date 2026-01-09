# OULAD Student Success Early Warning (Databricks + Spark + Deep Learning)

This project builds an **early-warning (at-risk) classifier** using the **Open University Learning Analytics Dataset (OULAD)**. The workflow is implemented in **Databricks** using a simple **Bronze → Gold** pattern:

- **Bronze**: raw CSV ingested into Delta tables (typed schemas)
- **Gold**: student-level feature table engineered for an “as-of day 28” early warning window
- **Models**:
  - Baseline: Logistic Regression
  - Deep Learning: Keras model with categorical embeddings + numeric features
- **Evaluation**:
  - AUC (ROC)
  - Confusion matrix + classification report
  - Threshold tuning for best F1
- **Tracking**: metrics and model artifacts logged to **MLflow**

---

## Table of Contents

1. [Project Goal](#project-goal)  
2. [Data Source](#data-source)  
3. [Databricks Architecture](#databricks-architecture)  
4. [Tables Created](#tables-created)  
5. [Features and Label](#features-and-label)  
6. [Splits and Evaluation Strategy](#splits-and-evaluation-strategy)  
7. [Modeling](#modeling)  
8. [Results Summary](#results-summary)  
9. [Visual Outputs](#visual-outputs)  
10. [How to Run (Databricks)](#how-to-run-databricks)  
11. [How to Make Predictions](#how-to-make-predictions)  
12. [Repository Structure](#repository-structure)  
13. [Links](#links)  

---

## Project Goal

The goal is to predict which students are **at risk** of **withdrawing or failing** early enough to intervene.  
This is implemented as a binary classification problem:

- **label = 1** → at-risk (Withdrawn or Fail)  
- **label = 0** → not at-risk (Pass or Distinction)

The “early warning” setup focuses on an **as-of day 28 window** (and can be adapted to day 14, etc.).

---

## Data Source

The project uses the OULAD dataset files (CSV), including:
- `assessments.csv`
- `courses.csv`
- `studentAssessment.csv`
- `studentInfo.csv`
- `studentRegistration.csv`
- `studentVle.csv`
- `vle.csv`

---

## Databricks Architecture

This project follows a lightweight medallion-style flow:

- **Volume storage** (raw CSV upload):  
  `dbfs:/Volumes/workspace/default/oulad_raw/`

- **Bronze schema** (typed Delta tables created from CSV):  
  `workspace.oulad_bronze`

- **Gold schema** (feature-engineered Delta tables):  
  `workspace.oulad_gold`

---

## Tables Created

### Bronze Delta Tables (raw → typed)
Created in: `workspace.oulad_bronze`
- `assessments`
- `courses`
- `studentAssessment`
- `studentInfo`
- `studentRegistration`
- `studentVle`
- `vle`

### Gold Feature Tables (student-level features)
Created in: `workspace.oulad_gold`
- `student_features_28d` (student-level features engineered from first 28 days)
- `student_features_28d_split` (stable student-level train/val/test split)
- `student_features_28d_time_split` (time-based split variant)

---

## Features and Label

### Label definition
A binary target label is derived from `studentInfo.final_result`:
- label = 1 if `final_result` ∈ {Withdrawn, Fail}
- label = 0 otherwise (Pass/Distinction)

### Feature families (examples)
- **Demographics / profile**: gender, region, highest_education, IMD band, age band, disability
- **Registration**: registration timing, unregistered-by-day28 indicator
- **VLE engagement (days 0–27)**: clicks, active days, unique sites, early/late ratio, short-window click counts
- **Assessment behavior (within window)**: submissions, average score, max score, weight submitted
- **Other engagement**: TMA/CMA submission counts in window

The Gold table stores one row per student (per module presentation) with all features aligned to the early-warning window.

---

## Splits and Evaluation Strategy

Two splitting strategies are used:

1. **Stable student-level split (random but consistent)**
   - Students are bucketed via hashing so the same student always lands in the same split:
     - Train ~80%
     - Val ~10%
     - Test ~10%

2. **Time-based split (realistic “future” validation)**
   - Splits by presentation/time so test more closely resembles deployment (train on earlier, test on later).

Both strategies are useful:
- Random split is good for rapid model iteration
- Time split is closer to a real early-warning deployment scenario

---

## Modeling

### Baseline: Logistic Regression
A baseline classifier is trained using:
- categorical handling (encoding)
- numeric scaling
- AUC evaluation + confusion matrix

### Deep Learning: Keras with categorical embeddings
The deep learning model:
- uses **one input per categorical feature**
- maps each categorical value to an **embedding vector**
- concatenates embeddings with scaled numeric features
- trains with callbacks (EarlyStopping, ReduceLROnPlateau)
- evaluates AUC, confusion matrix, classification report
- optionally tunes decision threshold to optimize F1

---

## Results Summary

Below are representative results from this notebook run.

### Time split
- **LR Val AUC:** 0.8897  
- **LR Test AUC:** 0.8964  
- **DL Val AUC:** 0.9065  
- **DL Test AUC:** 0.9101  

### Deep learning threshold tuning (example)
Best F1 threshold selected on validation:
- Best threshold by F1 ≈ **0.35–0.37**
- Reported best F1 ≈ **0.83–0.85** (varies by split)

Interpretation:
- AUC indicates strong ranking performance for early-warning prioritization
- Threshold tuning provides a controllable tradeoff between recall and precision depending on intervention capacity

---

## Visual Outputs

The notebook generates standard model visuals such as:
- ROC curve (test)
- Confusion matrix heatmap (selected threshold)
- Threshold curve (precision/recall/F1 vs threshold)

These visuals are intended for both:
- model evaluation
- stakeholder-facing interpretation (“what threshold should we use?”)

---

## How to Run (Databricks)

1. **Upload the CSVs to a Volume**
   - Destination: `workspace / default / <your_volume>`
   - Example: `dbfs:/Volumes/workspace/default/oulad_raw/`

2. **Run ingestion**
   - Read each CSV with explicit schema
   - Write to Bronze as Delta tables in `workspace.oulad_bronze`

3. **Build Gold feature table**
   - Create `workspace.oulad_gold.student_features_28d`

4. **Create splits**
   - Stable student-level split: `student_features_28d_split`
   - Time split (optional): `student_features_28d_time_split`

5. **Train + evaluate baseline and DL models**
   - Track metrics and artifacts using MLflow

---

## How to Make Predictions

For a “real” early-warning workflow:

1. Build features **as-of day 28** (or day 14, etc.) for the **current presentation**
2. Score only students who are currently active / newly registered
3. Use `pred_proba` to rank students for intervention:
   - “Top 10% highest risk” is often more operationally useful than a hard cutoff

Typical operational output:
- student_id / module / presentation
- risk probability
- risk band (e.g., Low/Medium/High)
- top contributing behaviors (optional, if you add explainability)

---

## Repository Structure

Suggested structure for this repo:

- `notebooks/`
  - `01_ingest_bronze.ipynb`
  - `02_build_gold_features.ipynb`
  - `03_train_models.ipynb`
- `reports/`
  - exported notebook HTML/PDF (optional)
- `images/`
  - ROC, confusion matrix, threshold curve screenshots (optional)
- `README.md`

---

## Links

> These are provided in code format so they copy cleanly.

```text
Kaggle dataset:
https://www.kaggle.com/datasets/anlgrbz/student-demographics-online-education-dataoulad

Databricks notebook:
https://dbc-8358d898-8164.cloud.databricks.com/editor/notebooks/932706482044283?o=7474646555686627
