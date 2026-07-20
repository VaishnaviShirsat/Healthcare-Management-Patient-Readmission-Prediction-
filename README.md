# 🏥Healthcare Management: Patient Readmission Prediction Analytics

> End-to-end data analytics capstone project analyzing 100k+ diabetic patient encounters to identify high-risk readmission patterns — built with **Excel, SQL (MySQL), Python, and Tableau**.


## 📌 Overview

**ReadmitGuard** is a simulated analytics engagement for a healthcare facility that wants to understand — and ultimately reduce — 30-day patient readmissions. The project takes a real-world diabetic patient encounters dataset through a complete analytics pipeline: exploratory analysis in Excel, relational data modeling and querying in SQL, statistical/EDA work in Python, and an interactive dashboard in Tableau.

**Business goal:** Identify high-risk patients and readmission drivers so healthcare providers can intervene proactively, improve patient outcomes, and reduce avoidable healthcare costs.

---

## 🗂️ Dataset

| | |
|---|---|
| **Source file** | `diabetic_data.csv` |
| **Records** | 101,766 unique patient encounters |
| **Columns** | 50 attributes |
| **Granularity** | One row per hospital encounter (patients can appear multiple times) |
| **Categories covered** | Demographics (race, gender, age, weight), admission details (type, source, discharge disposition), clinical metrics (lab procedures, medications, procedures, diagnoses, time in hospital), 23 medication columns, and the target variable `readmitted` (`<30`, `>30`, `NO`) |

> ⚠️ The dataset uses `?` as a placeholder for missing values across several fields (`weight`, `payer_code`, `medical_specialty`, diagnosis codes, etc.).

---

## 🧰 Tech Stack

| Layer | Tools Used |
|---|---|
| Data Exploration & Reporting | Microsoft Excel (Power Query, PivotTables, Data Analysis ToolPak) |
| Data Storage & Querying | MySQL 8.0 (MySQL Workbench), `LOAD DATA LOCAL INFILE` bulk import |
| Exploratory Data Analysis | Python 3.13 — `pandas`, `numpy`, `matplotlib`, `seaborn` |
| Dashboarding & Visualization | Tableau Public/Desktop |

---

## 1️⃣ Excel — Data Exploration

- Imported `diabetic_data.csv` via **Data → From Text/CSV**, then ran **Descriptive Statistics** (Data Analysis ToolPak) on key numeric fields.
- Built **PivotTables** for readmissions by gender/age group, race distribution, and readmission category counts.
- Built supporting **PivotCharts** (donut chart for race, bar chart for readmission counts).

**Key stats:** patients receive an average of **~43 lab tests** and **~16 medications** per encounter, with a typical hospital stay of **~4.4 days**.

**Key findings:**
- Females (54,708 encounters) slightly outnumber males (47,055) in the dataset.
- Both genders show a nearly identical **<30-day readmission rate (~11.2%)** — gender alone is not a strong predictor of early readmission.
- Race distribution is heavily skewed: **Caucasian patients make up ~75%** of the dataset, African American ~19%, with Asian, Hispanic, and Other combined under 6% — a notable class imbalance.
- Overall readmission split: **53.9% not readmitted, 34.9% readmitted after 30 days, 11.2% readmitted within 30 days** (the critical high-risk group).

---

## 2️⃣ SQL — Data Loading & Analysis (MySQL)

### Schema & Load
```sql
CREATE DATABASE Healthcare;
USE Healthcare;
SET GLOBAL local_infile = 1;

CREATE TABLE diabetic_data (
  encounter_id INT, patient_nbr INT, race VARCHAR(50), gender VARCHAR(20),
  age VARCHAR(20), weight VARCHAR(20), admission_type_id INT,
  discharge_disposition_id INT, admission_source_id INT, time_in_hospital INT,
  payer_code VARCHAR(20), medical_specialty VARCHAR(100),
  num_lab_procedures INT, num_procedures INT, num_medications INT,
  number_outpatient INT, number_emergency INT, number_inpatient INT,
  diag_1 VARCHAR(20), diag_2 VARCHAR(20), diag_3 VARCHAR(20), number_diagnoses INT,
  -- ...23 medication columns...
  change_col VARCHAR(10), diabetesMed VARCHAR(10), readmitted VARCHAR(10)
);

LOAD DATA LOCAL INFILE 'diabetic_data.csv'
INTO TABLE diabetic_data
FIELDS TERMINATED BY ',' ENCLOSED BY '"' LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```
> `local_infile` is enabled because MySQL Workbench blocks large local file loads by default — required to bulk-import all 101,766 rows reliably.

### Analysis Queries & Results

| Task | Result |
|---|---|
| Total patient encounters | **101,766** |
| Top diagnoses (`diag_1`) | Circulatory codes (e.g. `428`, `414`) dominate |
| Avg. length of stay by admission type | Ranges ~3.1–4.6 days depending on type |
| Readmitted patients | `<30`: 11,357 (11.16%) · `>30`: 35,545 (34.93%) · `NO`: 54,864 (53.91%) |
| Age distribution | Peaks in the **70–80** age bracket (26,068 encounters) |
| Most common procedures | The plurality of encounters had **0 procedures** performed |
| Avg. medications by age group | Increases with age, from ~6 (0–10) up to ~17 (60–70) |
| Readmission by payer code | Distribution varies notably by payer code, with self-pay/unknown codes showing different rates than insured groups |

Full query set covers: total encounter counts, top-10 diagnosis frequencies (`diag_1/2/3`), average stay by admission type, readmission rate breakdown, age distribution, procedure frequency, medication averages by age, and payer-code readmission distribution.

---

## 3️⃣ Python — Exploratory Data Analysis

**Libraries:** `pandas`, `numpy`, `matplotlib`, `seaborn`

**Analysis performed:**
- Descriptive statistics for all numeric features (`describe()`)
- Categorical distribution plots for `race` and `gender`
- Stacked bar chart: readmission status by age group
- Correlation heatmap across 8 numeric features
- Medication-change flag and total-medications distribution
- Diagnosis category classification (mapped ICD-9 codes → Circulatory, Respiratory, Digestive, Diabetes, Injury, Musculoskeletal, Genitourinary, Neoplasms, Other)
- Admission type/source/discharge disposition distributions
- Boxplots to flag outliers across 6 numeric columns

### Key Insights

- **Race:** Dataset dominated by one group (Caucasian) → potential fairness/bias risk for any downstream model.
- **Gender:** Near-even split, not a major differentiator.
- **Age vs. readmission:** The **70–80** bracket has the highest *absolute* number of readmissions; the **80–90** bracket shows a slightly *lower* <30-day rate proportionally, possibly reflecting different discharge/care practices for the oldest patients.
- **Strongest correlation:** `num_medications` ↔ `time_in_hospital` = **0.47** — longer stays are associated with more medications, consistent with clinically complex cases.
- **Weakest relationship:** `number_emergency` ↔ `num_lab_procedures` ≈ **0.00** — emergency cases tend to bypass extensive lab workups.
- **Diagnosis categories:** Circulatory conditions are by far the most frequent primary diagnosis category, followed by "Other" and Respiratory.
- **Admissions:** Most encounters originate from **emergency/urgent** admission types.
- **Outliers:** Present in `num_lab_procedures`, `num_medications`, and `number_emergency`, generally indicating more severe/complicated cases. Most patients stay a short 3–7 days.

### Suggested framing for a fraud/readmission risk report
When writing up findings (e.g., in the context of a fraud-detection or risk-scoring narrative), structure around:
1. **Dataset overview** — shape, dtypes, missingness
2. **Key statistics** — central tendencies for utilization metrics
3. **Demographics** — dominant race/gender segments
4. **Readmission patterns** — which age/admission segments over-index
5. **Correlations** — strongest inter-feature relationships
6. **Outliers** — which columns and their potential impact on model training
7. **Conclusion** — features most predictive of readmission risk (age, num_medications, time_in_hospital, number_inpatient, number_diagnoses stand out as candidates)

---

## 4️⃣ Tableau — Interactive Dashboard

Connected directly to `diabetic_data.csv` and built four linked worksheets combined into a single dashboard:

1. **Diagnoses vs. Readmissions** — bar charts of `Number Diagnoses` and `Number Emergency` by `Readmitted`.
   - `NO` group has the largest diagnosis volume (largest patient population).
   - `>30` days shows ~3× the diagnosis volume of `<30` days — reflecting chronic disease burden.
   - `<30` days has very few associated emergencies, suggesting many are **scheduled** readmissions rather than emergent ones.

2. **Age Bucket by Readmission** — calculated fields `Age Numeric` (`INT([Age])`) and `Age Bucket` (binned into 0–10 … 51+) crossed with `Readmitted` and count of `Number Inpatient`.
   - Diabetic inpatient cases are overwhelmingly concentrated in the **51+** bucket.
   - Confirms the 54% / 35% / 11% NO / >30 / <30 split at the inpatient level.

3. **Stacked Bar of Multiple Metrics** — `Num Medications`, `Num Procedures`, `Number Diagnoses`, `Number Emergency`, `Time in Hospital`, faceted by `Readmitted`.
   - Average medication count (~16) is consistently the dominant metric across all three readmission groups.
   - The `<30` day group trends slightly higher on average medications — a possible marker of case complexity.

4. **Bubble Chart** — `Num Medications` (x) × `Num Lab Procedures` (y), sized by `Number Diagnoses`, colored by `Readmitted`.
   - Clear separation: the `NO` group clusters at the highest medication/lab-procedure volume; `<30` day readmits cluster lowest.

All four views are combined into a single **Dashboard** for a unified, filterable view of readmission risk factors.

---

## 🔑 Cross-Tool Summary of Findings

| Theme | Finding |
|---|---|
| **Class balance** | Readmission target is imbalanced (54% / 35% / 11%) — important for any future ML modeling (consider resampling or class weighting). |
| **Demographic risk** | Age is a stronger signal than gender; risk concentrates in older adults (51+), peaking around 70–80. |
| **Clinical utilization** | `num_medications` and `time_in_hospital` are the most correlated numeric pair and likely useful predictive features. |
| **Care setting** | Emergency admissions dominate overall volume, but `<30`-day readmissions have low associated emergency counts, hinting some are planned. |
| **Data quality** | Significant missingness in `weight`, `payer_code`, and `medical_specialty` (`?` placeholders) needs handling before any modeling. |
| **Fairness consideration** | Race distribution is highly imbalanced (75% Caucasian) — flag for bias mitigation if this dataset feeds a predictive model. |

---

## 🚀 Next Steps (Suggested)

- Encode categorical variables and handle missing values (`?` → NaN, imputation or "Unknown" category).
- Engineer features: diagnosis category groupings, medication-change flags, age as numeric.
- Train a baseline classification model (e.g., logistic regression, XGBoost) to predict `readmitted` (binary: `<30` vs. rest).
- Address class imbalance (SMOTE / class weights) and evaluate with precision/recall/AUC rather than accuracy alone.
- Publish the Tableau dashboard to Tableau Public/Server for stakeholder access.

---

## 📄 License

This project is intended for educational/portfolio purposes. Dataset originally sourced from the UCI Machine Learning Repository ("Diabetes 130-US hospitals for years 1999–2008").

## 🙋 Author's Note

This repository documents a capstone-style project spanning the full analytics stack — Excel for quick stakeholder-friendly summaries, SQL for structured querying at scale, Python for deeper statistical EDA, and Tableau for interactive storytelling — mirroring a realistic healthcare analytics workflow.
