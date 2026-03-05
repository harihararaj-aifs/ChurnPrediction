# Phase 1 – ML Reporting Pipeline Architecture

## Overview

Phase 1 builds the initial ML-driven reporting system. Data is extracted nightly from operational databases, stored in cloud blob storage, processed using DuckDB for feature engineering, and used by ML models to generate predictive reports.

### Scope

* Churn Prediction
* Staffing Prediction
* Fraud Detection

---

# Architecture Flow

```mermaid
flowchart LR

subgraph DataSources
A[UC MySQL]
B[SiteWatch Firebird Rep-hub]
end

subgraph DataExtraction
C[Python Extraction<br>Initial + Daily Night Jobs<br>Tools: Python/Airbyte/dlt]
end

subgraph CloudStorage
D[GCP Blob Storage]
end

subgraph FeatureEngineering
E[DuckDB Feature Processing]
F[Feature Store<br>Local DuckDB]
end

subgraph MachineLearning
G[ML Models]
end

subgraph Predictions
H[Churn Prediction - Weekly]
I[Staffing Prediction - Daily]
J[Fraud Detection - Daily]
end

subgraph Reporting
K["DuckDB<br>(Storing Predictions back to the DuckDB for reporting and future queries)"]
end

A --> C
B --> C
C --> D
D --> E
E --> F
F --> G
G --> H
G --> I
G --> J
H --> K
I --> K
J --> K
```

---

# Pipeline Components

## Data Sources

Operational databases used by the car wash platform.

* UC MySQL
* SiteWatch Firebird

---

## Data Extraction

Python ETL jobs perform:

* Initial full extraction
* Daily incremental extraction
* Scheduled **nightly runs**

These jobs extract operational data from both databases.

---

## Cloud Storage

Extracted data is stored in:

**GCP Blob Storage**

Benefits:

* Cheap storage
* Decouples ingestion from analytics
* Enables reprocessing if needed

---

## Feature Engineering

DuckDB reads the extracted files from GCP Blob storage.

Tasks include:

* Data cleaning
* Aggregations
* Feature creation
* Behavioral metrics

The resulting **feature set is stored in a local DuckDB database**.

---

## Machine Learning Layer

ML models run on the engineered feature set.

Models predict:

* Customer churn probability
* Future staffing demand
* Fraud anomalies

---

## Outputs

The system generates prediction reports:

* **Churn Prediction Report**
* **Staffing Forecast Report**
* **Fraud Detection Report**

These reports can be scheduled or shared with stakeholders.
