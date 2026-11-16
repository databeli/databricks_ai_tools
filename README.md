# Payments Lakehouse

A Databricks lakehouse that measures the impact of a payment-gateway outage on an online retailer. The project generates synthetic source data, loads it through a bronze → silver → gold pipeline, exposes the results in a governed metric view, and shows them on a published dashboard.

Everything is defined in one Databricks Asset Bundle (`payments_lakehouse`) and runs on serverless compute.

## The scenario

The retailer takes payments through three gateways: **NimbusPay** (about 70% of traffic), **CedarPay** and **OrbitPay**. On **2026-09-24, 10:00–15:00 UTC**, NimbusPay failed about 60% of payment attempts, up from its normal ~2%. Orders failed and revenue was lost. Enterprise customers, about 5% of customers but about 55% of revenue, were hit hardest, and several of them downgraded their tier within the next 14 days.

The pipeline answers four questions:

1. How much revenue was lost?
2. Which gateway, regions and customers were affected?
3. Did those customers downgrade?
4. Did things recover?

## Architecture

```
Synthetic data generator (local Python, Databricks Connect)
        │  multi-part files
        ▼
/Volumes/payments_lakehouse/payments/landing      ← raw landing area
        │  incremental ingestion
        ▼
payments_bronze   raw rows + source file and arrival time
        │  expectations (drop / warn), order-loss check
        ▼
payments_silver   cleaned, typed, de-duplicated; customers as SCD Type 2
        │
        ▼
payments_gold     daily sales, hourly gateway health, customer outage impact, fact_orders
        │
        ├── payments_metrics          metric view (semantic layer)
        └── Payments Outage Impact    AI/BI dashboard
```

### Source data

About two months of skewed data with daily and weekly seasonality, written as files so the pipeline can pick them up incrementally:

| Source    | Contents                                                                 |
|-----------|--------------------------------------------------------------------------|
| Products  | Small product catalog                                                    |
| Customers | Change feed (inserts, updates, deletes) with region and tier             |
| Orders    | Line items, channel and amount                                           |
| Payments  | Gateway, attempt number, status and latency, including retries and refunds |

The data includes deliberate quality problems for the pipeline to catch: orders with a missing customer, a non-positive amount or a future date; duplicate orders; and unknown payment statuses. The generator can be re-run, and it can append one extra day of data as new files.

### Pipeline layers (`payments_medallion`)

- **Bronze:** incremental ingestion of each source, keeping where each row came from and when it arrived.
- **Silver:** cleaned, typed and de-duplicated data. Customer history is kept as SCD Type 2. Data quality expectations drop or flag bad rows, and the update fails if more than `max_order_loss_pct` of orders are lost between bronze and silver.
- **Gold:**
  - `daily_sales`: daily revenue and lost revenue
  - `gateway_health_hourly`: hourly attempts, failures and latency per gateway
  - `customer_outage_impact`: per-customer impact, using each customer's tier **at order time**, with revenue 14 days before and after and any downgrade
  - `fact_orders`: order-level fact table for the semantic layer

All transformations are SQL and use named parameters (`:param`) instead of hard-coded names.

### Pipeline parameters (dev)

| Parameter            | Value                                    |
|----------------------|------------------------------------------|
| `landing_path`       | `/Volumes/payments_lakehouse/payments/landing` |
| `bronze_schema`      | `payments_bronze`                        |
| `silver_schema`      | `payments_silver`                        |
| `gold_schema`        | `payments_gold`                          |
| `outage_start`       | `2026-09-24 10:00:00`                    |
| `outage_end`         | `2026-09-24 15:00:00`                    |
| `impact_window_days` | `14`                                     |
| `max_order_loss_pct` | `5`                                      |

### Semantic layer and dashboard

The metric view `payments_lakehouse.payments_gold.payments_metrics` defines:

- **Measures:** Revenue, Lost Revenue, Orders, Failed Orders, Payment Failure Rate, Affected Enterprise Customers
- **Dimensions:** date, hour, region, channel, gateway, tier

The **Payments Outage Impact** dashboard is built on the metric view and contains:

- KPI tiles for the main measures
- Hourly payment failure rate by gateway, from the day before the outage to the day after
- Lost revenue during the outage by region and tier
- Recovery trend: daily failure rate by gateway
- Top impacted enterprise customers, with downgrade status
- Filters for date, region and gateway

## Project layout

```
databricks.yml                         bundle definition (dev and prod targets)
requirements.txt                       local Python dependencies
PROJECT.md                             project brief
docs/
  raw_data_profile.md                  profile of the raw source data
resources/
  bronze.schema.yml, silver.schema.yml, gold.schema.yml
  payments_medallion.pipeline.yml      declarative pipeline
  payments_incremental_refresh.job.yml orchestration job
  payments_outage.dashboard.yml        dashboard resource
src/
  data_gen/generate.py                 synthetic data generator
  pipelines/payments_medallion/transformations/
    bronze/                            products_raw, customers_cdc_raw, orders_raw, payments_raw
    silver/                            products, customers_history, orders, payments, orders_reconciliation
    gold/                              daily_sales, gateway_health_hourly, customer_outage_impact, fact_orders
  semantic/payments_metrics.sql        metric view definition
  dashboards/payments_outage.lvdash.json
```

## Getting started

### Prerequisites

- A Databricks workspace with Unity Catalog and serverless compute
- A SQL warehouse
- Databricks CLI 1.12 or later, authenticated with a profile for the workspace
- Python 3.12, the version Databricks Connect 16.x and later expects

### Local environment

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
```

Dependencies: `databricks-connect` (>=16.4,<17.4), `faker`, `pandas`.

### Generate the data

```bash
python src/data_gen/generate.py
```

This writes the historical data to the landing volume. Run it again in its add-a-day mode to append one more day of files for an incremental load; see the script for its options.

### Validate and deploy

```bash
databricks bundle validate -t dev  -p <profile>
databricks bundle validate -t prod -p <profile>
databricks bundle deploy   -t dev  -p <profile>
```

Only the `dev` target is deployed. `prod` is validated but not deployed.

### Run

The `payments_incremental_refresh` job runs only when started manually. It has no schedule or file triggers, and it sends an email on failure. It runs two tasks:

1. `refresh_payments_medallion`: an incremental (not full-refresh) update of the pipeline
2. `publish_metric_view`: creates or replaces the `payments_metrics` metric view on the SQL warehouse

```bash
databricks bundle run payments_incremental_refresh -t dev -p <profile>
```

Because ingestion is incremental, each run processes only the files that arrived since the last run.

## Conventions

- Serverless compute everywhere. Local Python runs in the project virtual environment.
- A separate schema for each medallion layer. The base `payments` schema holds only the landing volume.
- Data quality is enforced only through pipeline expectations on the bronze → silver load. There are no separate check jobs.
- Ask before any full refresh, dropping tables or deleting resources.
