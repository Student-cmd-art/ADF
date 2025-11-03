# Azure Data Factory Pipeline Series

## Project Overview
This repository contains a collection of **Azure Data Factory (ADF)** pipelines I’ve built to demonstrate data engineering concepts.

## Pipeline 1: Change Data Capture (CDC)
This project implements a **Change Data Capture (CDC)** pipeline in **Azure Data Factory** that extracts only updated or new rows from a SQL table, stores them as Parquet files in **Azure Data Lake Storage Gen2**, and updates the CDC timestamp for the next run.

The setup also includes a **scheduler pipeline** that orchestrates the CDC process and triggers notification emails via a **Logic App** depending on success or failure.

---

## Pipeline Components

### 1. `pl_CDC`
Handles the incremental data extraction and CDC logic.

<img width="1051" height="381" alt="Screenshot 2025-11-03 at 10 01 16 AM" src="https://github.com/user-attachments/assets/3a569048-814c-404f-ba70-4f11545370c9" />


**Key steps:**
- **Lookup CDC Timestamp**  
  Reads the most recent `cdc_timestamp` from a JSON file stored in ADLS (e.g., `source/monitor/cdc.json`).
- **Query Source Data**  
  Uses the timestamp to filter rows from the SQL table where `last_updated > cdc_timestamp`.
- **IfCondition check**  
  Proceeds only if new rows are found (`TOTAL_COUNT > 0`).
- **Copy Data Activity**  
  Extracts qualifying rows from SQL and writes them as Parquet files to `sink/orders/` in ADLS.
- **Calculate Max CDC**  
  Determines the latest timestamp from the new data (`MAX(last_updated)`).
- **Update CDC File**  
  Overwrites the `cdc.json` file with the new timestamp for the next run.

**Parameters:**

| Parameter  | Type   | Description                                  |
|-------------|--------|----------------------------------------------|
| `schema`    | string | Source schema name                           |
| `table`     | string | Source table name                            |
| `backdate`  | string | Optional parameter to override CDC timestamp |

---

### 2. `pl_scheduled`
Controls scheduling and alerting logic.

<img width="545" height="381" alt="Screenshot 2025-11-03 at 10 01 47 AM" src="https://github.com/user-attachments/assets/aa3235df-f99e-4463-ab65-dfcd5316dbda" />


**Key steps:**
- Executes `pl_CDC` pipeline for a specific schema and table.  
- Uses an **IfCondition** to check if the pipeline succeeded or failed.  
- Sends results to a **Logic App** endpoint (HTTP POST) to trigger a success or failure email notification.  

---

## Linked Services

| Name                      | Type                 | Purpose                                         |
|----------------------------|----------------------|-------------------------------------------------|
| `AzureSqlDatabase1`        | Azure SQL Database   | Source system for data extraction               |
| `LS_SECONDSTORAGEACCOUNT`  | Azure Data Lake Gen2 | Storage for input/output JSONs and Parquet data |

All credentials and connection details are redacted. In a production setup, these would be securely stored and referenced via **Azure Key Vault**.

---

## Datasets

| Dataset        | Type    | Description                                    |
|----------------|---------|------------------------------------------------|
| `jsoncdc`      | JSON    | Holds current CDC timestamp                    |
| `ds_cdc`       | JSON    | Target dataset for writing new CDC timestamp   |
| `ds_empty`     | JSON    | Empty placeholder dataset used for CDC updates |
| `parquet_sink` | Parquet | Destination for incremental data      |

---

## How It Works (Simplified Flow)

1. `Lookup CDC Timestamp` → get last processed timestamp  
2. `Query SQL table` for rows where `last_updated` is newer  
3. If new data found → copy to ADLS as Parquet  
4. Calculate and update new `cdc_timestamp`  
5. Scheduler pipeline (`pl_scheduled`) triggers emails on success/failure  
