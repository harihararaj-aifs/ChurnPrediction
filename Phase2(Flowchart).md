# Phase 2 – Advanced Data Platform Architecture

## Overview

Phase 2 introduces a modern ELT pipeline using automated ingestion tools and a scalable analytical warehouse.

### Scope

* Additional analytics use cases
* Data warehouse
* Dashboards and chat interfaces

---

## Architecture Flow

```mermaid
flowchart LR

subgraph Sources
A[UC MySQL]
B[SiteWatch Firebird Replica]
end

subgraph Ingestion
C[Airbyte / dlt]
end

subgraph Storage
D[RAW / ODS Tables]
end

subgraph Processing
E[DuckDB]
F[DBT Transformations]
end

subgraph Warehouse
G[MotherDuck]
end

subgraph Analytics
H[Churn Report]
I[Staffing Report]
J[Fraud Report]
K[Dashboards / Chat]
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
G --> K
```

---

## Components

**Sources**

* UC MySQL
* SiteWatch Firebird replica

**Ingestion**

* Airbyte or dlt pipelines

**Storage Layer**

* RAW / ODS tables

**Processing**

* DuckDB used for compute
* dbt handles transformations

**Warehouse**

* MotherDuck for centralized analytics storage

**Consumption**

* Reports
* Dashboards
* Chat interfaces over analytics data
