# 🏥 Real-Time Hospital Patient Flow Analytics Pipeline (Azure + Databricks)

## 📌 Overview
This project implements a **real-time hospital patient flow analytics pipeline** using the **Bronze–Silver–Gold (BSG)** architecture on Microsoft Azure.  

The solution ingests streaming patient events, processes them with **Azure Databricks + Delta Lake**, orchestrates pipelines with **Azure Data Factory**, and models data into a **star schema** in **Azure Synapse Analytics** for reporting.  


## 🏗️ Architecture


---

## 🔑 Security
- **Azure Key Vault** used to securely manage all secrets (Event Hub keys, Storage keys, Synapse credentials).
- Connections to ADLS and Synapse configured via **Managed Identity** where possible.

---


## ⚙️ Pipeline Components

### 🏥 0. Data Generator
### 0. Data Generator
- A **Python script (`data_generator.py`)** simulates hospital patient events.  
- Uses **`kafka-python`** to connect securely to **Azure Event Hub** (Kafka endpoint) with **SASL/SSL** authentication.  
- Generates realistic event records:
  - `patient_id` (UUID)  
  - `gender` (Male/Female)  
  - `age` (1–100, with ~5% invalid ages >100 for dirty data simulation)  
  - `department` (Emergency, Surgery, ICU, Pediatrics, Maternity, Oncology, Cardiology)  
  - `admission_time` & `discharge_time` (UTC timestamps, with ~5% chance of future admission for dirty data)  
  - `bed_id` (1–500)  
  - `hospital_id` (1–7 hospitals in the network)  
- Runs continuously in a loop and **streams JSON messages to Event Hub every second**.  
- Purpose: provides a **continuous, semi-random, slightly “dirty” dataset** to test downstream Bronze → Silver → Gold processing.


---

### 🤎 1. Bronze Layer (Raw Ingestion)
- **Databricks Structured Streaming** reads from Event Hub (Kafka endpoint).  
- Stores raw JSON payloads in **Bronze Delta tables** in ADLS.  
- Append-only, schema-less → keeps all raw data for durability and lineage.  

---

### 🩶 2. Silver Layer (Cleansing & Structuring)
- Reads raw Bronze data with streaming.  
- Applies schema and cleanses:
  - Converts string timestamps to `TimestampType`.  
  - Corrects invalid/future admission times.  
  - Normalizes patient ages (invalid >100 → replaced).  
  - Ensures schema evolution with default columns.  
- Writes to **Silver Delta tables** in ADLS.  
- Provides **curated, query-ready data**.

---

### 💛 3. Gold Layer (Star Schema Modeling)
- Reads cleansed Silver data.  
- Builds **star schema** in Delta:

  **Dimensions**  
  - `Dim_Patient`: SCD Type 2, tracks patient attributes (gender, age) with `effective_from`, `effective_to`, `is_current`.  
  - `Dim_Department`: unique departments and hospital IDs.  

  **Fact**  
  - `Fact_Patient_Flow`: joins Silver with dimensions using surrogate keys.  
  - Calculates KPIs:
    - `length_of_stay_hours`
    - `is_currently_admitted`
    - Partitioned by `admission_date` for optimized queries.  

- Written to **Gold Delta tables** in ADLS.  

---

### 🏭 4. Orchestration with Azure Data Factory
- ADF pipeline orchestrates the workflow.  
- **Trigger condition**: fires when ≥5 new records are added to Silver.  
- Ensures Gold layer refreshes **only when new data is available**.  
- Configured alerts on pipeline failures.  

---

###  🏬 5. Azure Synapse Analytics (SQL Layer)

#### External Tables
- Defined external data source with **Managed Identity** → secure ADLS access.  
- Created **external tables** on Gold Delta outputs (via Parquet manifests):
    - Dimension Tables
        - `dim_patient`
        - `dim_department`
    - Fact Table
        - `fact_patient_flow`

#### KPI Views
Created SQL views to support reporting & Power BI:

- **`vw_bed_occupancy`** → % beds occupied (by gender)  
- **`vw_bed_turnover_rate`** → bed turnover (patients per bed)  
- **`vw_patient_demographics`** → patient counts by gender  
- **`vw_avg_treatment_duration`** → average stay (by department & gender)  
- **`vw_patient_volume_trend`** → daily patient inflow trend  
- **`vw_department_inflow`** → patient inflow by department  
- **`vw_overstay_patients`** → count of long-stay patients (>50h)  

---

## 🛠️ Tech Stack
- **Azure Key Vault** – secret management  
- **Azure Event Hub** – real-time event ingestion  
- **Azure Data Lake Storage Gen2** – Bronze/Silver/Gold containers  
- **Azure Databricks** – Structured Streaming, Delta Lake, SCD handling  
- **Azure Data Factory** – pipeline orchestration & triggers  
- **Azure Synapse Analytics** – external tables, SQL pools, views  
- **Power BI** (planned) – dashboards & visualization  

---