# BMW Cars Data Engineering

End-to-end Azure data engineering project for BMW car sales data using **Azure Data Factory (ADF)** for orchestration and ingestion, **Azure SQL Database** as source/control metadata, **ADLS Gen2** as Bronze storage, and **Azure Databricks** notebooks for Silver/Gold transformations.

## Project overview

This repository contains an ADF factory export and Databricks notebooks to implement a medallion-style pipeline:

- **Source preparation** pipeline ingests CSV data from GitHub into Azure SQL (`CarSales`).
- **Incremental load** pipeline reads watermark values, copies only new rows into ADLS Bronze parquet, and updates watermark.
- **Incremental + Databricks** pipeline extends the incremental load by running notebook-based Silver and Gold transformations.

## Repository structure

- `pipeline/`
  - `source_preparation.json`: Copies CSV from GitHub raw URL into SQL `CarSales` table.
  - `incremental_load.json`: Incremental copy from SQL to ADLS Bronze + watermark update.
  - `incremental_load_databricks_run.json`: Incremental copy + Databricks notebook chain for Silver/Gold.
- `dataset/`
  - `ds_git.json`: Parameterized HTTP delimited dataset for GitHub-hosted CSV files.
  - `sd_sqlDB.json`: Parameterized Azure SQL table dataset.
  - `ds_bronze.json`: ADLS Gen2 parquet dataset for Bronze layer (`bronze/BmwCarSales`).
- `linkedService/`
  - `ls_git_data.json`: Anonymous HTTP connection to `raw.githubusercontent.com`.
  - `AzureSqlDatabase1.json`: Azure SQL linked service.
  - `AzureDataLakeStorage1.json`: ADLS Gen2 linked service.
  - `AzureDatabrickslink.json`: Azure Databricks linked service.
- `databricks/`
  - `db_notebook.ipynb`: Library/bootstrap notebook.
  - `silver_notebook.ipynb`: Bronze ➜ Silver processing.
  - `gold_dim_branch.ipynb`, `gold_dim_date.ipynb`, `gold_dim_model.ipynb`, `gold_dim_dealer.ipynb`: Dimension builds.
  - `gold_fact_sales.ipynb`: Gold fact table build.
- `data/`
  - Sample sales files used by ingestion (`SalesData.csv`, `IncrementalSales.csv`).

## Pipeline flow

### 1) Source preparation (`source_preparation`)

1. Reads `IncrementalSales.csv` using `ds_git` from GitHub raw path.
2. Writes rows into SQL table `dbo.CarSales` using `sd_sqlDB`.

### 2) Incremental extraction (`incremental_load`)

1. `last_load` lookup reads watermark from `watermark_carsales`.
2. `current_load` lookup computes current max `Date_ID` in `CarSales`.
3. Copy activity extracts rows where `DATE_ID` is between last and current watermark and writes parquet to Bronze.
4. Stored procedure `dbo.updateWatermarkTable` updates watermark to current max date.

### 3) Transformations (`incremental_load_databricks_run`)

After the incremental copy and watermark update, ADF runs Databricks notebooks in sequence:

- `import-libs`
- `silver`
- parallel dimension notebooks: `branch_dim`, `date_dim`, `model_dim`, `dealer_dim`
- `fact` after all dimensions succeed

## Prerequisites

- Azure subscription with:
  - Azure Data Factory
  - Azure SQL Database
  - ADLS Gen2 storage account
  - Azure Databricks workspace + running cluster
- Permission to publish/import ADF factory JSON artifacts.
- SQL objects in source DB:
  - `dbo.CarSales`
  - `dbo.watermark_carsales`
  - stored procedure `dbo.updateWatermarkTable`

## Configuration checklist

Before deploying/running in your environment, update the linked services with your own values/secrets:

- SQL server/database and credentials in `linkedService/AzureSqlDatabase1.json`
- ADLS account URL and credentials in `linkedService/AzureDataLakeStorage1.json`
- Databricks domain/cluster/token in `linkedService/AzureDatabrickslink.json`

Also validate:

- `dataset/ds_git.json` relative GitHub raw path for your repo/branch.
- Notebook paths in `pipeline/incremental_load_databricks_run.json` (workspace path style differs across environments).

## How to run

1. Deploy/import this repo content to your ADF instance (factory export structure).
2. Confirm linked service connectivity.
3. Run `source_preparation` to seed/update `CarSales` in SQL.
4. Run `incremental_load` for Bronze-only incremental loads, or
5. Run `incremental_load_databricks_run` for full Bronze ➜ Silver ➜ Gold execution.

## Notes

- The repository stores ADF metadata as JSON and Databricks notebooks as `.ipynb`.
- Do not commit real credentials/tokens. Replace with Key Vault-backed secrets in production.

- Thanks for visiting :)
