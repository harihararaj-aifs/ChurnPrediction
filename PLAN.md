# Churn Prediction – Implementation Plan

## Project Goal

Build a machine learning system to predict customer churn for JAX car wash subscriptions. The system scores every active member (1–100 churn risk), predicts churn 30–90 days in advance, classifies churn type, and triggers automated retention actions.

---

## Churn Types in Scope

| Type | Description | Priority |
|---|---|---|
| Voluntary | Customer consciously cancels | High |
| Involuntary | Payment failure causes cancellation | High (20–40% of churn) |
| Passive | Usage drops, subscription still active | Medium |
| Early Tenure | Churns within first 1–2 months | Medium |
| Seasonal | Cyclical cancel/re-subscribe (Michigan winter/summer) | Medium |
| Downgrade | Revenue churn via plan downgrade | Low |
| Forced | Policy/fraud violation (not flagging yet per Jon) | Deferred |

---

## Deliverables

1. Nightly ETL pipeline extracting data from UC MySQL and SiteWatch Firebird
2. Feature store (DuckDB) with all engineered churn signals
3. Trained churn ML model (binary + multi-class)
4. Churn risk score (1–100) visible on every customer record
5. Weekly churn report with SHAP-based feature explanations
6. Automated retention action triggers (flags, offers, campaigns)

---

## Implementation Phases

---

### Phase 1 – ML Reporting Pipeline *(Build this first)*

**Goal**: End-to-end working pipeline with churn predictions and weekly reports.

#### Step 1 – Data Extraction

- Connect to UC MySQL and SiteWatch Firebird (Rep-hub)
- Write Python ETL scripts:
  - **Initial full extraction** of all historical data
  - **Daily incremental extraction** (nightly scheduled job)
- Tables to extract:

| Table | Purpose |
|---|---|
| Sale | Total invoice per transaction |
| SalePasses | Membership sales details |
| SalesPayment | Credit card payments per transaction |
| Customer | Customer details (jax_unlimited_club) |
| PlanType / PlanTypeType | Plan configuration |
| SaleCharges | Charge records |
| SaleItems | Line items per sale |
| SALEEMPLOYEES | Employee associations |
| jax_access | Employee details |

- Store extracted files as **CSV/Parquet in GCP Blob Storage**
- Staging pattern: multiple CSVs in GCP → pull into downstream store

#### Step 2 – Feature Engineering (DuckDB)

Read raw files from GCP Blob Storage into DuckDB. Build the following feature groups:

**Usage / Visit Behavior**
- `washes_last_7d`, `washes_last_30d`, `washes_last_90d`
- `days_since_last_wash`
- `usage_trend` = `washes_last_30d - washes_prev_30d`
- `usage_momentum` = `(washes_last_30d - washes_prev_30d) - (washes_prev_30d - washes_prev_60d)`
- `max_gap_days_last_90d`
- `preferred_location`, `location_switch_count`
- `weekday_weekend_ratio`
- `relative_usage` = `washes_last_30d / avg_washes_for_all_customers_in_same_month`

**Subscription State**
- `plan_tier` (basic / premium)
- `monthly_price`
- `tenure_months`
- `plan_changes_count`
- `auto_renew` flag
- `pause_suspend_status`

**Payment & Billing**
- `payment_failures_last_30d`, `payment_failures_last_90d`
- `retry_count_last_bill`
- `days_since_last_successful_payment`
- `in_dunning` flag
- `card_expiring_soon` flag
- `outstanding_balance`, `chargeback_flag`
- `consecutive_payment_failures`
- `first_time_failure_flag`
- `retry_count_trend`

**Customer Experience**
- `complaints_last_90d`
- `refund_count`
- `rewash_request_count`

**Seasonality**
- `month`, `week_of_year`
- `is_winter`, `is_spring`, `is_summer` flags (Michigan calendar: Nov–Mar = busy)

**Store**
- `primary_store_id` (store customer visits most)

Store the final feature set in a **local DuckDB feature store table** (`customer_features`).

#### Step 3 – Label Generation

Define churn labels for supervised training:

- **Binary label**: Did customer churn within the next 30 days? (0 = No, 1 = Yes)
- **Multi-class label**: 0 = No Churn, 1 = Voluntary Churn, 2 = Involuntary Churn
  - Voluntary: customer-initiated cancellation
  - Involuntary: subscription lapsed due to payment failure after retries
- **High-risk passive flag**: `auto_renew = TRUE` AND `days_since_last_wash > 45`
- **Early tenure flag**: `tenure_months < 3`

Label generation requires careful historical window construction (avoid data leakage — features must be computed from data available *before* the label window).

#### Step 4 – Model Training

**Recommended model**: XGBoost or LightGBM (handles tabular data, missing values, class imbalance well)

Training steps:
1. Train/validation/test split (time-based split — not random, to prevent leakage)
2. Handle class imbalance: `scale_pos_weight` or SMOTE
3. Tune hyperparameters (Optuna or grid search)
4. Evaluate: AUC-ROC, Precision-Recall, F1 per churn type
5. Seasonal calibration: validate that summer predictions aren't over-penalized vs. winter baseline

**Churn risk score**: `score = round(churn_probability × 100)`

**Explainability**: Use SHAP to output top feature drivers per prediction:
- Usage drop patterns
- Payment failures
- Seasonal context
- Site experience signals
- Multi-site usage (hypothesis: multi-site users churn less)

#### Step 5 – Prediction Storage

Write model outputs to local DuckDB tables:

- `churn_predictions` — customer_id, score, churn_type, predicted_churn_date, top SHAP features, run_date
- `staffing_predictions` — (separate model, out of scope here)
- `fraud_detections` — (separate model, deferred)

#### Step 6 – Reporting

- **Weekly Churn Report**: ranked list of high-risk customers, churn type breakdown, top SHAP drivers
- Distribute to stakeholders (format TBD: email, dashboard, CSV)

#### Step 7 – Retention Action Triggers

Automated actions based on score thresholds:

| Segment | Score | Action |
|---|---|---|
| Critical | 80–100 | Flag for MSA intervention + save offer |
| High Risk | 60–79 | Re-engagement campaign |
| Involuntary Risk | Any payment failure signal | SMS card update reminder |
| Passive | High score + no wash 45d | Targeted incentive |

---

### Phase 2 – Advanced Data Platform *(After Phase 1 is stable)*

**Goal**: Replace the Phase 1 batch pipeline with a scalable, modern ELT architecture.

#### Components

| Layer | Tool | Purpose |
|---|---|---|
| Ingestion | Airbyte or dlt | Automated ELT from MySQL + Firebird |
| Storage (RAW) | MotherDuck | RAW and ODS tables |
| Transformation | dbt + DuckDB | Modular SQL transformations |
| Warehouse | MotherDuck | Centralized analytics warehouse |
| Analytics | Data marts | Churn, Staffing, Fraud marts |
| Interface | Dashboards + MCP chat | MotherDuck MCP for agentic queries |

#### Key Decisions

- Write in larger batches to MotherDuck (per MotherDuck recommendation)
- Stage data in GCP as CSV/Parquet, then bulk load into MotherDuck
- Add column-level descriptions/comments in dbt schema.yml for documentation
- Attach agentic chat layer to MotherDuck via MCP server

#### dbt Model Layers

```
RAW (source tables, untouched)
  └── Staging (light cleaning, renamed columns)
        └── Intermediate (joins, aggregations)
              └── Marts (churn_features, churn_scores, churn_reports)
```

---

## Feature Flags / Decisions Pending

| Decision | Status | Notes |
|---|---|---|
| Fraud detection | Deferred | Per Jon — not flagging frauds yet |
| Forced churn labeling | Deferred | Waiting on policy definition |
| Survival analysis (when will they churn) | Phase 2+ | Spotify-style, needs more data |
| Google Reviews integration | Future | Site-level signal for experience |
| GLACCOUNT table usage | Open | Clarify with team |
| CUSTOMERCODE table | Open | Clarify with team |
| Multiple MySQL databases | Open | Question for Dexter |

---

## Seasonal Considerations (Michigan)

- **Nov–Mar**: Busiest season — model must not under-predict churn during this period
- **Summer**: Lower usage baseline — avoid over-predicting churn for customers who always slow down in summer
- Mitigation: use `relative_usage` (usage vs. month average) instead of raw wash counts

---

## Success Metrics

| Metric | Target |
|---|---|
| AUC-ROC (binary churn) | > 0.80 |
| Precision at top 10% risk | > 60% |
| Involuntary churn recall | > 75% |
| Score visible on every active member record | Yes |
| Weekly report delivered on schedule | Yes |
| Retention action trigger latency | < 24 hours |

---

## Tech Stack Summary

| Component | Phase 1 | Phase 2 |
|---|---|---|
| Ingestion | Python ETL scripts | Airbyte / dlt |
| Storage | GCP Blob Storage (CSV/Parquet) | MotherDuck (RAW/ODS) |
| Feature Engineering | DuckDB (local) | DuckDB + dbt |
| ML Framework | XGBoost / LightGBM | Same + Survival Analysis |
| Explainability | SHAP | SHAP |
| Feature Store | Local DuckDB | MotherDuck mart |
| Reporting | Python-generated CSV/email | Dashboards + MCP chat |
| Orchestration | Cron / simple scheduler | TBD (Airflow / Prefect) |

---

## Open Questions

1. What is the exact retry logic and dunning steps in the current billing system?
2. How is `auto_renew = FALSE` recorded — is it a status column or an event log?
3. Is there a `pause/suspend` status on subscriptions?
4. What is the definition of "cancelled" in the UC MySQL schema (status code)?
5. What is in the GLACCOUNT and CUSTOMERCODE tables?
6. Why are there multiple databases in MySQL? (Question for Dexter)
7. Is card expiry date accessible in SalesPayment or a related table?
