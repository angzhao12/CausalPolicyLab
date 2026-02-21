## Causal Incrementality & Policy Optimization (Starbucks Rewards)

### Executive Summary
This project implements an end-to-end **causal incrementality + targeting policy** system designed to minimize promotional waste in marketing campaigns.  
Unlike propensity models that predict purchase probability, this system estimates **incremental impact (uplift)**—identifying users whose behavior changes because of an intervention—and translates uplift scores into **profit-aware targeting decisions**.

### Business Problem: Optimizing Marketing Spend
In large-scale loyalty programs, issuing discounts to “Sure Things” (users who would purchase anyway) results in significant subsidy waste. We apply **quad-tree segmentation** to categorize users by heterogeneous treatment response:

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
pip install -r requirements.txt

Download the Starbucks dataset and place it under data/starbucks/raw/ (see data/starbucks/README.md).

Run notebooks in order:

01_build_dataset.ipynb

02_uplift_model.ipynb

03_policy_simulator.ipynb

Key Results (TBD)
Model	AUUC	Qini	Top-10% Uplift
T-Learner	TBD	TBD	TBD
LinearDML	TBD	TBD	TBD
CausalForestDML	TBD	TBD	TBD

Simulated policy impact:

Profit increase: TBD% vs random targeting

Subsidy waste reduction: TBD% by excluding Sure Things

Limitations & Roadmap

Observational bias: estimates rely on unconfoundedness assumptions; future work includes an RCT-based track (e.g., Criteo).

Productionization: refactor notebook logic into modular Python scripts for deployment-ready pipelines.