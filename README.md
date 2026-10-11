# Semiconductor Datasheet Pipeline

> An AI-powered data pipeline that ingests semiconductor datasheet PDFs, extracts structured specifications using LLMs, and delivers analytics-ready tables to a Power BI dashboard. Built on Databricks Free Edition.

---

## Overview

This project demonstrates an end-to-end data engineering workflow:

- **Ingest** unstructured PDF datasheets into Unity Catalog Volumes
- **Parse** text with PyMuPDF and extract specifications using Databricks `ai_query()` with Llama 3.3 70B
- **Transform** through a medallion architecture (Bronze → Silver → Gold)
- **Validate** with a quarantine pattern that preserves all records while flagging quality issues
- **Visualize** in Power BI Desktop via a live Databricks SQL Warehouse connection
- **Package** the entire pipeline as a Databricks Asset Bundle (DAB) for reproducible deployments

---

## Architecture

```
PDF Datasheets                              Databricks                                   Power BI
──────────────                              ──────────                                   ────────
                                                                                       
  Volume    →   Bronze (raw text)   →   Silver (AI-extracted)   →   Gold (business-ready)   →   Dashboard
                PyMuPDF ingestion         ai_query() + JSON parse    Normalized + enriched         Live SQL
                                                                              │
                                                                              ▼
                                                                   Quarantine (quality rules)
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Storage | Databricks Unity Catalog, Delta Lake, Volumes |
| Processing | PySpark, SQL, PyMuPDF |
| AI | Databricks `ai_query()` with Llama 3.3 70B Instruct |
| Orchestration | Databricks Workflows |
| Deployment | Databricks Asset Bundles (DAB), Databricks CLI |
| Version Control | Git, GitHub |
| Visualization | Power BI Desktop (Import mode, SQL Warehouse connector) |

---

## Repository Structure

```
semiconductor-datasheet-pipeline/
├── README.md
├── notebooks/                              # Databricks Repos-synced notebooks
│   ├── 01_bronze_ingest.ipynb
│   ├── 02_silver_extract.ipynb
│   ├── 03_gold_build.ipynb
│   └── 04_quality_checks.ipynb
└── semiconductor_pipeline/                 # Databricks Asset Bundle
    ├── databricks.yml                      # Bundle config + targets
    ├── resources/
    │   └── semiconductor_pipeline.job.yml  # Job definition
    └── notebooks/                          # Bundle-deployed notebooks
```

---

## Data Flow

| Layer | Table | Purpose |
|---|---|---|
| Bronze | `datasheet_project.bronze.datasheets_raw` | Raw PDF text per file |
| Silver | `datasheet_project.silver.datasheets_extracted` | AI-extracted JSON specs |
| Silver | `datasheet_project.silver.datasheets_clean` | Parsed into typed columns |
| Silver | `datasheet_project.silver.datasheets_quarantine` | Quality-tagged source of truth |
| Gold | `datasheet_project.gold.chip_catalog` | Normalized, enriched catalog |
| Gold | `datasheet_project.gold.chip_catalog_clean` | Clean records view (dashboard source) |
| Gold | `datasheet_project.gold.chip_catalog_quarantine` | Failed records view (remediation) |

---

## Data Quality Approach

The pipeline implements the **quarantine pattern** using inverse logic:

- All records flow into a single Silver table with an `is_quarantined` boolean flag.
- Records failing any rule (negative values, suspicious ratios, invalid voltage ranges, etc.) are flagged with a human-readable `quarantine_reason`.
- Two Gold views split the data: one for analytics, one for remediation.
- **Guarantee:** `clean + quarantined = total`. Zero data loss.

Example rules enforced:
- `ram_kb <= flash_kb * 10` catches AI misreads of memory units
- `voltage_min < voltage_max` catches logical inconsistencies
- Range checks on clock speed and pin count

---

## Deployment

The pipeline is packaged as a Databricks Asset Bundle. Deploy to a development workspace:

```powershell
cd semiconductor_pipeline
databricks bundle validate
databricks bundle deploy
databricks bundle run semiconductor_pipeline_main
```

---

## Dashboard

A Power BI dashboard connects live to the Gold layer via the Databricks SQL Warehouse connector. Current visuals:

- Flash memory by chip (bar)
- Chip count by architecture family (donut)
- Max clock speed by chip (bar)
- Memory profile: flash vs RAM (scatter)
- Slicers for manufacturer, chip name, architecture family, and flash range

The `.pbix` file is included in the repository.
