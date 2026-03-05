# Phase 1 – Data Pipeline Architecture

## Overview

Phase 1 focuses on building the initial reporting platform using Python extraction pipelines and DuckDB for analytics.

### Scope

* Churn Report
* Staffing Report
* Fraud Report

---

## Architecture Flow

```mermaid
flowchart LR

subgraph Sources
A[UC MySQL]
B[SiteWatch Firebird Replica]
end

subgraph Extraction
C[Python Extract<br>Initial + Daily]
end

subgraph Processing
D[GCP VM]
E[DuckDB]
end

subgraph Reports
F[Churn Report<br>Weekly]
G[Staffing Report<br>Daily]
H[Fraud Report<br>Daily]
end

A --> C
B --> C
C --> D
D --> E
E --> F
E --> G
E --> H
```

---

## Components

**Sources**

* UC MySQL
* SiteWatch Firebird replica database

**Ingestion**

* Python scripts perform initial and daily extraction.

**Compute**

* Runs on a GCP VM.

**Storage / Query Engine**

* DuckDB used for analytics and report generation.

**Outputs**

* Weekly Churn Report
* Daily Staffing Report
* Daily Fraud Report
