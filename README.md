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
  Reads the most recent `cdc_timestamp` from a JSON file stored in ADLS (e.g., `source/monitor/cdc_timestamp.json`).
- **Script to Query Source Data**  
  Uses the timestamp to filter rows from the SQL table where `last_updated > cdc_timestamp`.
- **IfCondition check**  
  Proceeds only if new rows are found (`TOTAL_COUNT > 0`).
- **Copy Data Activity**  
  Extracts qualifying rows from SQL and writes them as Parquet files to `sink/orders/` in ADLS.
- **Script to Calculate Max CDC**  
  Determines the latest timestamp from the new data (`MAX(last_updated)`).
- **Copy Acticity to Update CDC File**  
  Overwrites the `cdc_timestamp.json` file with the new timestamp for the next run.

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

## Pipeline 2: REST API Data Extraction (Pokémon API)

### Overview
The **second pipeline** in this series demonstrates how to extract data from a **public REST API**, handle **pagination dynamically**, and store the output as JSON files in **Azure Data Lake Storage Gen2**.

It connects to the **PokéAPI** (a public dataset for Pokémon information) to fetch data in batches using an offset-based pagination rule.

---

### Pipeline Components

#### `pl_API`
Implements the main extraction logic using three key activities:

<img width="809" height="163" alt="Screenshot 2025-11-03 at 10 17 17 AM" src="https://github.com/user-attachments/assets/a41c2f1d-5766-4355-a6f1-84e69262181b" />


**1. Get Pokemon Data**  
- A **Web Activity** sends a GET request to the PokéAPI endpoint to fetch the total number of Pokémon (`count`).  
- Example API call: `https://pokeapi.co/api/v2/pokemon`

**2. Set Count of Output**  
- Stores the total record count in a pipeline variable named `count`.  
- This value is later used for pagination for fetching all records.

**3. Copy Pokemon Data with Offset**  
- A **Copy Data** activity that uses `RestSource` with pagination rules.  
- The `QueryParameters.{offset}` property is dynamically generated using:
  ```text
  RANGE:0:@{variables('count')}:20
 which loops through all records in increments of 20.
- Each API call fetches a batch of Pokémon data and writes it to ADLS as JSON.

- Note: you can also use the AbsoluteURL on the next value in the response body for pagination
  
<img width="1115" height="463" alt="Screenshot 2025-11-03 at 10 17 50 AM" src="https://github.com/user-attachments/assets/61f42868-f17f-40cf-8cca-33ab1a056df6" />

---

## Linked Services

| Name                      | Type                 | Purpose                                         |
|----------------------------|----------------------|-------------------------------------------------|
| `ls_pokemon`        | REST Service   | Connects to the public PokéAPI endpoint              |
| `LS_SECONDSTORAGEACCOUNT`  | Azure Data Lake Gen2 | Storage for input/output JSONs |

---

## Datasets

| Dataset        | Type    | Description                                    |
|----------------|---------|------------------------------------------------|
| `ds_pokemon`      | REST Resource    | Represents the source API resource                  |
| `ds_json`       | JSON    | Output dataset storing results in ADLS   |

---


