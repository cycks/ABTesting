# A/B Testing Analysis: Client Retention Loyalty Program

This repository contains the statistical analysis notebook for evaluating the **Loyalty Points & Public Recognition Program**. The experiment tests whether introducing a loyalty program (Treatment) causally increases the **90-Day Client Retention Rate** compared to standard onboarding (Control).

---

## 📋 Table of Contents
1. [Experiment Overview](#experiment-overview)
2. [Hypothesis Formulation](#hypothesis-formulation)
3. [Environment Setup & Dependencies](#environment-setup--dependencies)
4. [Step-by-Step Notebook Analysis](#step-by-step-notebook-analysis)
   - [Cell 1: Database Setup & Hypothesis Definition](#cell-1-database-setup--hypothesis-definition)
   - [Cell 2: Data Extraction (Intent-to-Treat)](#cell-2-data-extraction-intent-to-treat)
   - [Cell 3: Power Analysis & Sample Size Verification](#cell-3-power-analysis--sample-size-verification)
   - [Cell 4: Sample Ratio Mismatch (SRM) Integrity Check](#cell-4-sample-ratio-mismatch-srm-integrity-check)
   - [Cell 5: Primary Metric Evaluation (Two-Proportion Z-Test)](#cell-5-primary-metric-evaluation-two-proportion-z-test)
   - [Cell 6: Guardrail Metric Check & Final Decision Framework](#cell-6-guardrail-metric-check--final-decision-framework)
5. [Summary of Results & Decision](#summary-of-results--decision)

---

## 🎯 Experiment Overview

* **Domain:** Client Retention in Professional Services / Consulting.
* **Target Metric (Primary):** 90-Day Retention Rate ($p$).
* **Guardrail Metric:** Average Revenue Per Client (ARPC).
* **Target Allocation:** 50% Control vs. 50% Treatment.
* **Database Engine:** PostgreSQL (`ABTesting` database).

---

## 🔬 Hypothesis Formulation

Let:
* $p_C$ = True 90-day retention rate for the **Control Group** (Standard Onboarding).
* $p_T$ = True 90-day retention rate for the **Treatment Group** (Loyalty Points & Public Recognition).

* **Null Hypothesis ($H_0$):** $p_T - p_C \le 0$  
  * *Business Meaning:* The loyalty program does not increase 90-day client retention. Any observed difference is due to random chance.
* **Alternative Hypothesis ($H_1$):** $p_T - p_C > 0$  
  * *Business Meaning:* The loyalty program causally increases 90-day client retention.

---

## 💻 Environment Setup & Dependencies

The analysis is implemented in Python within a Jupyter Notebook environment. The required libraries are:

```bash
pip install numpy pandas psycopg2-binary scipy statsmodels
```

---

## 📑 Step-by-Step Notebook Analysis

### Cell 1: Database Setup & Hypothesis Definition
* **Purpose:** Sets up module imports (`pandas`, `scipy`, `statsmodels`, `psycopg2`), documents the experimental hypotheses, and configures the PostgreSQL database connection parameters (`DB_CONFIG`).
* **Function Defined:** `get_db_connection()` — Establishes and returns an active connection to the `ABTesting` PostgreSQL instance.

---

### Cell 2: Data Extraction (Intent-to-Treat)
* **Purpose:** Resolves selection bias and confounding noise by querying randomized experiment records directly from the database.
* **Function Defined:** `extract_experiment_data(conn)` — Executes a SQL `JOIN` across `clients`, `experiment_assignments`, and `client_activity` tables to extract:
  - `client_id`
  - `variant` (Control vs. Treatment)
  - `retained_90d` (1 = Retained, 0 = Churned)
  - `total_revenue`
  - `milestones_completed`
* **Output:** Loads 200,000 randomized records into a Pandas DataFrame (`df`) and displays the first 10 rows.

---

### Cell 3: Power Analysis & Sample Size Verification
* **Purpose:** Resolves sampling error and prevents Type I/II errors by calculating the statistical power requirements.
* **Function Defined:** `verify_sample_size(p1=0.40, mde=0.06, alpha=0.05, power=0.80)` — Computes the required sample size per variant using `statsmodels.stats.api.NormalIndPower`.
* **Key Inputs:**
  - Baseline Retention ($p_1$): $40.0\%$
  - Minimum Detectable Effect (MDE): $+6.0\%$
  - Significance Level ($\alpha$): $0.05$
  - Statistical Power ($1 - \beta$): $0.80$
* **Result:** Confirms a minimum requirement of **1,068 clients per group** (2,136 total). The extracted dataset ($N = 200,000$) significantly exceeds this threshold, ensuring high statistical power.

---

### Cell 4: Sample Ratio Mismatch (SRM) Integrity Check
* **Purpose:** Verifies randomization integrity to detect tracking failures or allocation bias before analyzing primary metrics.
* **Function Defined:** `check_sample_ratio_mismatch(df)` — Runs a **Chi-Square Goodness-of-Fit Test** (`scipy.stats.chisquare`) comparing observed group counts against the expected 50/50 split.
* **Output:**
  - Observed Split: **Control:** 99,926 | **Treatment:** 100,074
  - $p$-value: $0.7407$
  - **Verdict:** $p \ge 0.01$. No SRM detected; randomization integrity is confirmed.

---

### Cell 5: Primary Metric Evaluation (Two-Proportion Z-Test)
* **Purpose:** Evaluates whether the treatment produced a statistically significant lift in 90-day retention.
* **Function Defined:** `evaluate_primary_metric(df, alpha=0.05)` — Computes the two-sided Z-test for two proportions (`statsmodels.stats.api.proportions_ztest`) and constructs a 95% Confidence Interval for the lift.
* **Results:**
  - **Control Retention Rate:** $39.73\%$
  - **Treatment Retention Rate:** $46.98\%$
  - **Observed Absolute Lift:** $+7.25\%$ (95% CI: $[+6.82\%, +7.68\%]$)
  - **Z-Score:** $32.7121$
  - **$p$-value:** $1.05 \times 10^{-234}$
* **Verdict:** $p < 0.05$. The retention lift is statistically significant.

---

### Cell 6: Guardrail Metric Check & Final Decision Framework
* **Purpose:** Ensures the treatment did not cause unintended financial harm, and executes the final business decision rule.
* **Function Defined:** `evaluate_guardrail_metrics(df, alpha=0.05)` — Performs a **Welch’s t-test** (`scipy.stats.ttest_ind(..., equal_var=False)`) on Average Revenue Per Client (ARPC) to handle potential variance heterogeneity.
* **Guardrail Results:**
  - **Control Mean Revenue:** $\$5,516.18$
  - **Treatment Mean Revenue:** $\$5,502.99$
  - **Revenue $p$-value:** $0.2553$
  - **Verdict:** No statistically significant drop in revenue ($p \ge 0.05$). Guardrail check passed.

---

## 📊 Summary of Results & Decision

| Evaluation Stage | Test Performed | Statistic / Metric | Result / Output | Status |
| :--- | :--- | :--- | :--- | :--- |
| **Power Calculation** | Normal Power Analysis | Min Required $N$ | 1,068 / group |  Passed |
| **SRM Integrity** | Chi-Square Goodness-of-Fit | $p$-value | 0.7407 |  Passed |
| **Primary Metric** | Two-Proportion Z-Test | Absolute Lift ($p$-value) | $+7.25\%$ ($1.05 \times 10^{-234}$) |  Significant Lift |
| **Guardrail Metric** | Welch's t-test | ARPC Difference ($p$-value) | $-\$13.19$ ($0.2553$) |  No Harm Detected |

### Final Business Recommendation
```text
==============================================================================
FINAL BUSINESS DECISION FRAMEWORK
==============================================================================
 STATISTICAL DECISION: Reject Null Hypothesis (H0).
 ACTIONABLE RECOMMENDATION: Roll out the Loyalty Points & Recognition Program
                             to 100% of consulting clients.
==============================================================================
```
