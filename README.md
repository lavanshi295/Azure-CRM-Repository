# Azure CRM Sales Data Pipeline

![Azure](https://img.shields.io/badge/Azure-Data%20Engineering-0078D4?style=for-the-badge&logo=microsoft-azure)
![ADF](https://img.shields.io/badge/Azure%20Data%20Factory-Orchestration-0078D4?style=for-the-badge&logo=microsoft-azure)
![Databricks](https://img.shields.io/badge/Databricks-PySpark-FF3621?style=for-the-badge&logo=databricks)
![Status](https://img.shields.io/badge/Status-Completed-brightgreen?style=for-the-badge)

An end-to-end Azure data engineering pipeline for **CRM sales opportunity data**. Data from an on-premises SQL Server is ingested through Azure Data Factory (via a Self-Hosted Integration Runtime), landed in Azure Data Lake Storage Gen2, cleaned and transformed with PySpark in Azure Databricks, and monitored with automated email alerts through Azure Logic Apps.

<p align="center">
  <img src="architecture/architecture_diagram.png" alt="Architecture Diagram" width="900"/>
</p>

---

## Business Problem

Sales teams track accounts, products and deals across multiple operational tables. This data usually lives in on-premises systems, is inconsistent, and is hard to analyse in one place.

This project builds an automated, cloud-based pipeline that:

- Moves CRM data from an on-premises SQL Server to the cloud securely
- Stores raw data untouched, so it can always be reprocessed
- Cleans and standardises the data at scale using PySpark
- Notifies the team automatically when a pipeline run succeeds or fails

---

## Flow at a Glance

The numbers below match the numbered badges in the architecture diagram above.

| Step | Stage | What happens | Azure service |
|:---:|---|---|---|
| 1 | Source | CRM tables live in an on-premises SQL Server | SQL Server |
| 2 | Ingestion | Data is copied securely to the cloud through a Self-Hosted Integration Runtime | Azure Data Factory |
| 3 | Raw storage | Files land untouched in the `raw-data` container | ADLS Gen2 |
| 4 | Transformation | Data is cleaned and transformed with PySpark | Azure Databricks |
| 5 | Curated storage | Analytics-ready output is saved in the `transformed-data` container | ADLS Gen2 |
| 6 | Orchestration alert | The pipeline's success or failure triggers a workflow | Azure Logic Apps |
| 7 | Notification | An email is sent to the team with the pipeline status | Outlook / Email |

Azure Key Vault supports steps 2 and 4 by holding all credentials, so nothing sensitive is stored in code.

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| Source | SQL Server (on-premises) | Operational CRM data |
| Connectivity | Self-Hosted Integration Runtime | Secure bridge between on-prem and Azure |
| Orchestration | Azure Data Factory | Pipeline scheduling and data movement |
| Version control | Git / GitHub (ADF Git integration) | Source control for ADF artifacts |
| Storage | Azure Data Lake Storage Gen2 | `raw-data` and `transformed-data` containers |
| Transformation | Azure Databricks, PySpark | Cleaning and transformation |
| Secrets | Azure Key Vault + Databricks secret scope | No hardcoded credentials |
| Alerting | Azure Logic Apps | Email notifications on pipeline success or failure |

---

## Dataset

The project uses a CRM sales opportunities dataset made up of five tables:

| File | Description |
|---|---|
| `accounts.csv` | Customer company accounts |
| `products.csv` | Product catalogue |
| `sales_pipeline.csv` | Sales opportunities and their deal progress |
| `sales_teams.csv` | Sales agents, managers and regional offices |
| `data_dictionary.csv` | Column definitions for the dataset |

> The full dataset is not included in this repository. A small sample is available in `data/sample/`. Original dataset source: *add link here*.

<p align="center">
  <img src="docs/screenshots/01_sql_server_source_tables.png" alt="Source tables in SQL Server" width="800"/>
  <br/><em>Step 1 - Source CRM tables in the on-premises SQL Server</em>
</p>

---

## Pipeline Walkthrough

### Step 2 - Ingestion with Azure Data Factory

- A **Self-Hosted Integration Runtime** is installed on the machine hosting SQL Server, so ADF can reach on-premises data without exposing the database publicly.
- A **Copy Data** activity moves each table from SQL Server into the `raw-data` container of ADLS Gen2.
- Linked service credentials are stored in **Azure Key Vault**.
- The ADF workspace is connected to **GitHub** for version control.

<p align="center">
  <img src="docs/screenshots/02_self_hosted_ir_running.png" alt="Self-Hosted Integration Runtime running" width="800"/>
  <br/><em>Self-Hosted Integration Runtime registered and running</em>
</p>

<p align="center">
  <img src="docs/screenshots/03_adf_pipeline_canvas.png" alt="ADF pipeline canvas" width="800"/>
  <br/><em>ADF pipeline with Copy Data activities</em>
</p>

<p align="center">
  <img src="docs/screenshots/04_adf_pipeline_run_success.png" alt="ADF pipeline run succeeded" width="800"/>
  <br/><em>Successful pipeline run in ADF Monitor</em>
</p>

### Step 3 - Raw Storage in ADLS Gen2

- `raw-data` holds an untouched copy of the source data, kept for reprocessing and auditing.

<p align="center">
  <img src="docs/screenshots/05_adls_raw_data_container.png" alt="raw-data container in ADLS Gen2" width="800"/>
  <br/><em>Files landed in the raw-data container</em>
</p>

### Step 4 - Transformation with Azure Databricks

Notebooks read directly from ADLS using the `abfss://` protocol, with credentials pulled from Key Vault through a secret scope:

```python
storage_account = "<your-storage-account>"

spark.conf.set(
    f"fs.azure.account.key.{storage_account}.dfs.core.windows.net",
    dbutils.secrets.get(scope="<your-scope>", key="<your-secret-key>")
)

accounts_df = (
    spark.read.format("csv")
    .option("header", "true")
    .option("inferSchema", "true")
    .load(f"abfss://raw-data@{storage_account}.dfs.core.windows.net/accounts.csv")
)
```

Transformations applied (edit to match your notebook):
- [ ] Data type corrections
- [ ] Null and duplicate handling
- [ ] Column standardisation
- [ ] Joins across accounts, products, pipeline and sales teams
- [ ] Output written to the `transformed-data` container

<p align="center">
  <img src="docs/screenshots/06_databricks_notebook.png" alt="Databricks transformation notebook" width="800"/>
  <br/><em>PySpark notebook reading raw data and applying transformations</em>
</p>

### Step 5 - Curated Storage in ADLS Gen2

- `transformed-data` holds the cleaned, analytics-ready output from Databricks.

<p align="center">
  <img src="docs/screenshots/07_adls_transformed_data_container.png" alt="transformed-data container in ADLS Gen2" width="800"/>
  <br/><em>Transformed output saved in the transformed-data container</em>
</p>

---

## Security

- No credentials are hardcoded anywhere in this repository
- Secrets are stored in **Azure Key Vault** and read in Databricks through a **secret scope**
- The on-premises database is never exposed publicly; access goes through the Self-Hosted Integration Runtime
- Sensitive values in this repo are replaced with placeholders such as `<your-storage-account>`

---

## Monitoring and Alerts

### Steps 6 and 7 - Logic App and Email Notification

A Logic App is called from the ADF pipeline on success and on failure, and sends an email so issues are caught without manually checking the portal.

| Event | Action |
|---|---|
| Pipeline succeeds | Logic App sends a success email |
| Pipeline fails | Logic App sends a failure email |

<p align="center">
  <img src="docs/screenshots/08_logic_app_workflow.png" alt="Logic App workflow" width="800"/>
  <br/><em>Logic App workflow that sends the status email</em>
</p>

<p align="center">
  <img src="docs/screenshots/09_email_alert.png" alt="Pipeline status email" width="800"/>
  <br/><em>Email notification received after a pipeline run</em>
</p>

---

## Repository Structure

```
azure-crm-sales-data-pipeline/
│
├── README.md
├── .gitignore
│
├── architecture/
│   └── architecture_diagram.png
│
├── databricks/
│   └── <your exported notebooks>
│
├── adf/
│   └── pipeline.json
│
├── logic-app/
│   └── workflow.json
│
├── data/
│   └── sample/
│
└── docs/
    └── screenshots/
        ├── 01_sql_server_source_tables.png
        ├── 02_self_hosted_ir_running.png
        ├── 03_adf_pipeline_canvas.png
        ├── 04_adf_pipeline_run_success.png
        ├── 05_adls_raw_data_container.png
        ├── 06_databricks_notebook.png
        ├── 07_adls_transformed_data_container.png
        ├── 08_logic_app_workflow.png
        └── 09_email_alert.png
```

---

## How to Reproduce

**Prerequisites:** Azure subscription, SQL Server (Express works), GitHub account.

1. **Prepare the source:** load the CRM tables into a local SQL Server database.
2. **Create the storage account:** ADLS Gen2 with `raw-data` and `transformed-data` containers.
3. **Create Key Vault:** store the SQL Server password and storage account key as secrets.
4. **Set up ADF:** create the factory, install the Self-Hosted IR on the SQL Server machine, and register it with the ADF authentication key.
5. **Build the pipeline:** create linked services (SQL Server, ADLS), datasets, and a Copy Data activity.
6. **Create the Databricks workspace:** create a secret scope backed by Key Vault, attach a cluster, and run the transformation notebooks.
7. **Add alerts:** create a Logic App with an HTTP trigger and an email action, then call it from ADF on success and failure.

---

## Challenges and Solutions

| Challenge | Solution |
|---|---|
| ADF could not reach the on-premises SQL Server | Installed and registered a Self-Hosted Integration Runtime |
| Reusing the IR across a second ADF workspace | Registered the existing IR node with a new authentication key |
| `dbutils.fs.mount()` failed ("not whitelisted") on serverless compute | Replaced mounts with direct `abfss://` access using account key credentials from a secret scope |
| Serverless compute rejected the storage config | Created a classic all-purpose cluster for the notebooks |
| Gmail connector failed in Logic Apps (insufficient authentication scopes) | Switched to the Outlook / Office 365 connector |
| Illegal character in `abfss://` URI | Corrected missing `f` prefix on f-strings used in file paths |

---

## Author

**Lavanshi**
Based in Delhi, India

[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?style=flat&logo=linkedin)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-black?style=flat&logo=github)](https://github.com/mayank2705)
