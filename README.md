# Data Engineering Reference

**From raw data to production pipelines — source systems, star schemas, and cloud infrastructure on GCP.**

Data engineering is the foundation of every analytics and AI system. Without clean, modeled, accessible data, dashboards show wrong numbers, ML models train on garbage, and business decisions are based on guesses.

This repository is a practitioner's reference for building production data pipelines — organized progressively from foundations to cloud-scale deployment.

---

## Concepts

Structured guides covering architecture and design decisions. Mermaid diagrams render automatically on GitHub.

### Data Modeling

| Chapter | Start Here |
|:---|:---|
| [Why Data Modeling Matters](concepts/Data_Modeling/01_Why.md) | The 3 AM call center story — why OLTP and OLAP are different |
| [Concepts](concepts/Data_Modeling/02_Concepts.md) | Star schema, dimensions, facts, surrogate keys, SCD |
| [Hello World](concepts/Data_Modeling/03_Hello_World.md) | Build a star schema on BigQuery — upload, model, query |

### Star Schema Design

| Chapter | What It Covers |
|:---|:---|
| [Why Star Schema](concepts/Star_Schema_Design/01_Why.md) | 7 problems with querying source tables directly |
| [The Design](concepts/Star_Schema_Design/02_Design.md) | Every dimension and fact table, with DDL |
| [Source Tables](concepts/Star_Schema_Design/02a_Source_Tables.md) | What source systems look like — schemas, relationships, the join pain |
| [Building It](concepts/Star_Schema_Design/03_Building_It.md) | Step-by-step BigQuery build with 6 reports + data quality checks |
| [Pipeline Process](concepts/Star_Schema_Design/04_Pipeline_Process.md) | End-to-end: ingest (GCS, Cloud Functions, MongoDB, SFTP) → Silver → Gold → serve (Looker, API, ML) + Cloud Composer orchestration |

---

## Notebooks

Hands-on Jupyter notebooks. Click any Colab badge to open and run.

### Foundations

| Notebook | Open in Colab |
|:---|:---|
| [DE Foundations](notebooks/M01_DE_Foundations.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M01_DE_Foundations.ipynb) |
| [Git, Linux & SQL Foundations](notebooks/M02_Git_Linux_SQL_Foundations.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M02_Git_Linux_SQL_Foundations.ipynb) |
| [Python for Data Engineering](notebooks/M03_Python_for_Data_Engineering.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M03_Python_for_Data_Engineering.ipynb) |
| [Advanced SQL](notebooks/M04_Advanced_SQL_for_DE.ipynb) — window functions, CTEs, optimization | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M04_Advanced_SQL_for_DE.ipynb) |

### Data Modeling & Architecture

| Notebook | Open in Colab |
|:---|:---|
| [Data Modeling](notebooks/M05_Data_Modeling.ipynb) — star schema, SCD, partitioning | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M05_Data_Modeling.ipynb) |
| [Distributed Systems](notebooks/M06_Distributed_Systems_Hadoop_to_EMR.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M06_Distributed_Systems_Hadoop_to_EMR.ipynb) |
| [PySpark](notebooks/M07_PySpark.ipynb) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M07_PySpark.ipynb) |

### Cloud Pipelines (GCP)

| Notebook | Open in Colab |
|:---|:---|
| [Cloud Data Pipeline Build](notebooks/M08_Cloud_Data_Pipeline_Build.ipynb) — GCS, BigQuery, Dataflow, Bronze/Silver/Gold | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M08_Cloud_Data_Pipeline_Build.ipynb) |
| [Pipeline Orchestration](notebooks/M09_Cloud_Data_Pipeline_Orchestrate.ipynb) — Airflow, Cloud Composer | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M09_Cloud_Data_Pipeline_Orchestrate.ipynb) |
| [CI/CD for Data Engineering](notebooks/M10_CICD_for_Data_Engineering.ipynb) — Terraform, GitHub Actions | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/M10_CICD_for_Data_Engineering.ipynb) |

### Pipeline Practicals

| Notebook | What It Does | Open in Colab |
|:---|:---|:---|
| [Analytics Pipeline](notebooks/Analytics_Pipeline.ipynb) | Bronze → Silver → Gold in Pandas (local, no cloud) | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/Analytics_Pipeline.ipynb) |
| [GCP Setup & Queries](notebooks/GCP_Pipeline.ipynb) | GCS + BigQuery setup, IAM, billing, raw queries | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/GCP_Pipeline.ipynb) |
| [**GCP Full Pipeline**](notebooks/GCP_Full_Pipeline.ipynb) | **Complete Bronze → Silver → Gold on BigQuery** — the star schema built end-to-end with data quality gates | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/GCP_Full_Pipeline.ipynb) |
| [ML Pipeline](notebooks/ML_Pipeline.ipynb) | Where DE output goes — feature engineering → model training → SHAP | [![Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sunilmogadati/data-engineer-accelerator/blob/main/notebooks/ML_Pipeline.ipynb) |

---

## The Dataset

All pipelines use a **call center analytics dataset** — synthetic data modeled on production DRTV (Direct Response Television) operations.

| File | What It Contains |
|:---|:---|
| `calls.json` | 510 call records (NDJSON) — timestamps, duration, disposition, nested customer data |
| `campaigns.csv` | 10 campaigns — DNIS → client, channel (VA/Live) |
| `orders.csv` | 78 orders — call_id, products, revenue |
| `products.csv` | 20 products — SKU, price, category |

**Intentional data quality issues** built into the dataset: duplicate records, timezone inconsistencies, orphaned orders, missing values. These mirror real production problems.

The companion [AI Engineering Reference](https://github.com/sunilmogadati/ai-engineer-accelerator) builds ML models on this same data — the DE pipeline feeds the AI pipeline.

---

## GCP Services Used

| Service | Role in the Pipeline | AWS Equivalent |
|:---|:---|:---|
| GCS (Cloud Storage) | Raw file storage (Bronze layer) | S3 |
| BigQuery | Data warehouse (Silver + Gold layers) | Redshift / Athena |
| Cloud Functions | Webhook ingestion, scheduled extracts | Lambda |
| Cloud Composer | Pipeline orchestration (Airflow DAGs) | MWAA / Step Functions |
| Cloud Scheduler | Cron triggers for batch jobs | EventBridge |
| Looker Studio | Dashboards connected to BigQuery | QuickSight |
| IAM | Access control — who can query what | IAM |

---

## Running Locally

```bash
git clone https://github.com/sunilmogadati/data-engineer-accelerator.git
cd data-engineer-accelerator
python3 -m venv .venv && source .venv/bin/activate
pip install pandas numpy matplotlib jupyter
jupyter notebook
```

For GCP notebooks: install [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) and authenticate with `gcloud auth login`.

Or click any Colab badge above — GCS and BigQuery CLI are pre-installed on Colab.

---

## Author

**Sunil Mogadati** — 25+ years building and leading complex systems end-to-end. Architect-Developer. Cloud, Data & AI. US Patent No. 10,638,891.

[LinkedIn](https://linkedin.com/in/sunilmogadati) · [Community](https://www.skool.com/workwise) · [GitHub](https://github.com/sunilmogadati)
