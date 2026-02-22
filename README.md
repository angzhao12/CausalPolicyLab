## Causal Incrementality & Policy Optimization (Starbucks Rewards)

### Executive Summary
This project implements an end-to-end **causal incrementality + targeting policy** system designed to minimize promotional waste in marketing campaigns.  
Unlike propensity models that predict purchase probability, this system estimates **incremental impact (uplift)**—identifying users whose behavior changes because of an intervention—and translates uplift scores into **profit-aware targeting decisions**.

### Business Problem: Optimizing Marketing Spend
In large-scale loyalty programs, issuing discounts to “Sure Things” (users who would purchase anyway) results in significant subsidy waste. I apply **quad-tree segmentation** to categorize users by heterogeneous treatment response:

- **Persuadables**: buy only if treated → primary target  
- **Sure Things**: buy regardless of treatment → exclude to reduce subsidy waste  
- **Lost Causes**: unlikely to buy either way → deprioritize  
- **Sleeping Dogs**: may react negatively to treatment → strictly exclude  

### System Architecture (3-Notebook Pipeline)
The system is structured as a sequential notebook pipeline:

#### `01_build_dataset.ipynb` — Causal Design
- Transforms raw event streams into a user-level causal table **(X, T, Y)**
- Leakage control via strict time windows:
  - **Baseline features** engineered from a pre-treatment lookback window (e.g., 30-day history)
  - **Outcomes** computed strictly within the offer validity window
- “Gold cohort” (Sample A): users exposed to the most common **BOGO** offer

**Outputs (generated locally):** gold-standard user table (not committed to GitHub).

#### `02_uplift_model.ipynb` — Decision-Focused Modeling & Evaluation
- Models:
  - Baseline: **T-Learner**
  - Causal ML: **EconML** (e.g., LinearDML, CausalForestDML)
- Metrics (decision-oriented):
  - **Qini curve**, **AUUC**, **Top-K uplift**
- Score direction calibration: higher score = higher targeting priority

**Outputs (generated locally):** uplift evaluation plots and calibrated targeting scores (not committed by default).

#### `03_policy_simulator.ipynb` — From Scores to Actions
Converts model outputs into actionable business strategies under constraints:

- Profit objective:
  \[
  \text{inc\_profit} = \text{uplift} \times \text{margin} - \text{coupon\_cost}
  \]
- Targeting policies:
  - **Threshold policy**: target only users with positive expected incremental profit
  - **Budget/Top-K policy**: maximize total profit under a fixed marketing budget (optional knapsack extension)
- Segment insights: compare Persuadables vs Sure Things across demographics (e.g., age/income)

**Outputs (generated locally):** policy profit summary and segment-level diagnostics (not committed by default).

### How to Run
1) Install dependencies:
```bash
pip install -r requirements.txt
```
2.  **Data Acquisition**: Download the Starbucks dataset and place it under data/starbucks/raw/ (see data/starbucks/README.md).
3.  **Execution Order**: Run notebooks `01` → `02` → `03`.

01_build_dataset.ipynb

02_uplift_model.ipynb

03_policy_simulator.ipynb


## Key Results (Filled with Actual Outputs)

### Uplift Model Performance (Ranking Quality)
I evaluate uplift models as a **ranking problem** (who to target first), using **Top-K incremental lift** and **Qini/AUUC** rather than classification accuracy.

| Model | AUUC (approx.) | Notes |
|---|---:|---|
| **T-Learner** | **~2115** | Strong baseline; performed best on this dataset slice. |
| **LinearDML (EconML)** | **~1261** | Required **rank calibration** (direction flip) to align “higher score = better to target.” |
| **CausalForestDML (EconML)** | **~1089** | Positive AUUC without flip in the best config, but still below T-Learner here. |

**Top-K takeaway:** On this ~5k sample slice, the T-Learner baseline is very competitive; DML variants add methodological rigor (orthogonalization / cross-fitting), but do not automatically outperform without careful evaluation and tuning.

---

### Policy Simulator: Profit-Aware Targeting Results
I translate predicted incremental spend into incremental profit using:

\[
\text{inc\_profit} = \text{uplift} \times \text{margin} - \text{coupon\_cost}
\]

Business assumptions used in the simulator:
- **MARGIN = 0.30**
- **COUPON_COST = 2.00**
- Example budget scenario: **BUDGET_TOTAL = 500**

#### Policy A — Threshold (Profit Gate Only)
**Rule:** target if `inc_profit > 0`

- **Coverage:** 77.87% (1295 / 1663 users)  
- **Total incremental profit:** **$3,466.56**  
- **Total cost:** **$2,590.00**  
- **ROI (profit / cost):** **1.3384**

Interpretation: high reach, decent ROI, but not maximizing efficiency because it includes all profit-positive users.

#### Policy B — Budget-Constrained Top-K (with Profit Gate)
**Rule:** keep `inc_profit > 0`, then choose top-K under budget  
- With **$500 budget** and **$2 cost**, **K = 250**

- **Targeted users:** **250**
- **Total incremental profit:** **$1,370.08**
- **Total cost:** **$500.00**
- **ROI:** **2.7402**

Interpretation: lower total profit than Policy A (fewer users targeted), but **much higher ROI** by spending only on the highest-value users.

---

### Segment & Business Insight (Quad-Tree)
I segment users by **Uplift rank × Baseline propensity rank** (0.5/0.5 cut in rank space):

- **Persuadables:** high uplift, low baseline → classic incentive-responsive group  
- **Sure Things:** low uplift, high baseline → avoid subsidizing natural buyers  
- **Sleeping Dogs:** low/negative uplift, high baseline → risky / wasteful  

In the budgeted run, spend concentrates on **High-High + Persuadables**, while **Sure Things and Sleeping Dogs are fully avoided**—consistent with the “reduce promo waste” narrative.

---

### Credibility Checks (Notebook03)
To ensure uplift ranking reflects real signal (not noise) and to diagnose bias risks:

- **Permutation placebo (shuffle treatment):**
  - Actual AUC-like summary: **1503.0628**
  - Placebo median AUC: **-16.8213**
  - Interpretation: strong evidence ranking captures real signal.

- **Imbalance diagnostics + IPW correction:**
  - Balance checks show the most extreme top-% slices can be imbalanced on baseline spend/income.
  - IPW diagnostics: treat rate **0.527**, PS range **0.397–0.867**, weight cap (p99) **2.89**
  - IPW-adjusted lift curve is lower than naive but remains positive across most thresholds → stronger credibility for reporting.


### Limitations & Roadmap

* Observational bias: estimates rely on unconfoundedness assumptions; future work includes an RCT-based track (e.g., Criteo).

* Productionization: refactor notebook logic into modular Python scripts for deployment-ready pipelines.
