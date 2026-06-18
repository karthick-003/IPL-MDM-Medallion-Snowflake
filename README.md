# IPL Data Engineering Pipeline (Medallion Architecture)

A production-ready ELT (Extract, Load, Transform) data pipeline built in **Snowflake** utilizing the **Medallion Architecture** (`Bronze` -> `Silver` -> `Gold`). The pipeline ingests raw Indian Premier League (IPL) delivery data, executes complex data cleaning, standardizes historical franchise rebrandings, and builds high-performance aggregation layers optimized for downstream analytics and reporting.

---

## 📊 Pipeline Architecture Overview

The system is split into three distinct logical schemas inside the `IPL_MDM` database to ensure strict separation of concerns, data quality enforcement, and auditability.

[ Raw CSV File ]
│
▼  (Inbound Ingestion via COPY INTO)
┌──────────────────────────────────────────┐
│ BRONZE LAYER (Raw Ingestion)             │ -> Append-only schema-on-read mirror
│   └── RAW_DELIVERIES                     │
└──────────────────────────────────────────┘
│
▼  (Data Cleansing & Transformation)
┌──────────────────────────────────────────┐
│ SILVER LAYER (Enriched / Conformed)      │ -> Normalization, surrogate keys,
│   └── CLEAN_DELIVERIES                   │    and historical team mapping
└──────────────────────────────────────────┘
│
▼  (Business Logic & Aggregations)
┌──────────────────────────────────────────┐
│ GOLD LAYER (Analytics / Presentation)    │ -> Fact/Dimension analytics ready
│   ├── TEAM_PERFORMANCE                   │
│   ├── BATTER_PERFORMANCE                 │
│   └── BOWLER_PERFORMANCE                 │
└──────────────────────────────────────────┘

---

## 💾 Dataset Information

Because the raw IPL ball-by-ball dataset exceeds GitHub's web upload limits, the data is hosted externally. 

* **Source:** [Kaggle - IPL Complete Dataset (2008-2020) by Patrickb1912](https://www.kaggle.com/datasets/patrickb1912/ipl-complete-dataset-20082020)
* **Ingestion Method:** Files are loaded directly into an external cloud landing zone/stage before execution of the Snowflake copy commands.

### Source Schema Overview
The dataset contains two primary data sources that act as the foundation for our Medallion Pipeline:
1. `IPL Matches 2008-2020.csv` — Contains high-level match summaries, venues, dates, toss decisions, and match winners.
2. `IPL Ball-by-Ball 2008-2020.csv` — Deeply detailed, granular ball-by-ball event data used for running our complex Silver transformation and Gold analytics layer.

---

## 🛠️ Key Engineering Implementations

* **Defensive SQL Architecture:** Implemented `NULLIF()` constraints across all metric division calculations (such as Strike Rates, Batting Averages, and Bowling Averages) to completely eliminate the risk of runtime **`Division-by-Zero`** errors.
* **Historical Franchise Standardization:** Handled structural changes and rebrandings in IPL history using conditional logic (`CASE WHEN`) to map legacy franchises into modern entities (e.g., merging *Deccan Chargers* into *Sunrisers Hyderabad*, *Delhi Daredevils* into *Delhi Capitals*, and *Kings XI Punjab* into *Punjab Kings*).
* **Granular Surrogate Key Generation:** Engineered a unique composite business identifier (`BALL_ID`) at the lowest atomic grain of the data by hashing/concatenating `MATCH_ID`, `INNING`, `OVER_NUMBER`, and `BALL`.
* **Data Integrity Auditing:** Integrated a unified `UNION ALL` data layer row-count auditing script to trace and monitor system balance across the entire lifecycle.

---

## 🗂️ Data Layers & Schema Design

### 1. Bronze Layer (`IPL_MDM.BRONZE`)
Acts as the initial landing zone for raw data. It mirrors the structure of the incoming source file exactly without modifying any values.
* **Inbound Stage:** `IPL_MDM.BRONZE.IPL_STAGE`
* **Target Table:** `RAW_DELIVERIES`
* **Ingestion Strategy:** Bulk-loaded via the `COPY INTO` command, handling structural `NULL` values like `'NA'`, `'N/A'`, or empty fields gracefully.

### 2. Silver Layer (`IPL_MDM.SILVER`)
Applies data cleansing, text normalization, and structural transformations:
* Normalizes the 0-indexed `OVER` column to standard match notation via `OVER + 1`.
* Converts null or empty string extras to an explicit, clean `'NO_EXTRA'` indicator.
* Enforces strict uppercase standardization on categorical attributes like `EXTRAS_TYPE` and `DISMISSAL_KIND`.
* Fills empty field strings for `PLAYER_DISMISSED` and `FIELDER` with descriptive placeholders (`'NOT_DISMISSED'`, `'NOT_APPLICABLE'`).

### 3. Gold Layer (`IPL_MDM.GOLD`)
Aggregated, read-optimized tables designed to serve business intelligence dashboards or direct analytical queries:
* `TEAM_PERFORMANCE`: Compiles total runs scored, wickets lost, matches played, and calculates the true average runs scored per match.
* `BATTER_PERFORMANCE`: Evaluates batting profiles including total runs, balls faced, strike rates, career averages, and single-ball highest scores.
* `BOWLER_PERFORMANCE`: Focuses on defensive bowling metrics including total balls delivered, runs conceded, wickets taken, strike speeds, and career bowling averages.

---

## 🚀 Deployment & Execution Guide

### Prerequisites
* A Snowflake account with access to a virtual warehouse (e.g., `COMPUTE_WH`).
* A role with privileges to create databases and schemas (e.g., `SYSADMIN`).
* The raw data file (`deliveries.csv`) uploaded to your configured internal stage.

### Execution Steps
1. Open your Snowflake Web UI (**Snowsight**).
2. Create a new SQL Worksheet and name it `ipl_mdm_medallion_pipeline.sql`.
3. Copy the complete pipeline script into the worksheet.
4. Highlight and run the sections in chronological order:
    * **Section 1 & 2:** Initializes your data architecture environments and executes the raw data ingestion copy job.
    * **Section 3:** Processes the raw rows into the clean, transformed Silver layer.
    * **Section 4:** Generates the aggregated analytics dashboards inside the Gold layer.
5. Scroll to the very bottom (**Section 5**) and execute the **Pipeline Audit Section** to return a clear, unified view of row balances across all structural data layers.
