# Project: Payments Lakehouse

## How to work on this project

- Work **one step at a time** and **stop after each step** until I say `continue`.
- After each step, report what was created or changed, plus a short checklist so I can verify it in the Databricks UI.
- Use the Databricks skills for *how* to build things. This brief only says *what* to build.
- Ask before any full refresh, dropping tables, or deleting resources.
- Before starting, ask me for the workspace details: CLI profile, catalog, base schema, SQL warehouse, alert email.

### Conventions

- Serverless compute everywhere. Local Python runs in a project virtual environment.
- Pipeline transformations in SQL, with named parameters (`:param`) instead of hard-coded names.
- A separate schema for each medallion layer (bronze, silver, gold). The base schema only holds the raw landing area.
- Data quality enforced only with pipeline expectations on the bronze → silver load. No separate check jobs.
- Jobs run manually and email someone on failure. No schedules or file triggers.
- Everything is defined in one bundle with a `dev` and a `prod` target. Validate both, deploy only `dev`.

## Business context

An online retailer takes payments through three gateways, one of which carries most of the traffic. On one day, that main gateway had a multi-hour outage: its failure rate jumped from ~2% to ~60%. Orders failed, revenue was lost, and enterprise customers were hit hardest. Some of them downgraded their tier in the following two weeks.

The data team needs a reliable pipeline that answers four questions:
- How much revenue was lost?
- Which gateway, regions and customers were affected?
- Did those customers downgrade?
- Did things recover?

## Source data (synthetic)

Generate about two months of realistic, skewed data with daily and weekly seasonality, written as multi-part **files** (not tables) so the pipeline can ingest them incrementally:

- **Products:** a small catalog.
- **Customers:** a change feed with inserts, updates and deletes, including region and tier.
- **Orders:** line items, channel and amount.
- **Payments:** gateway, attempt number, status and latency, including retries and refunds.

The data must tell the story: enterprise customers are a small share of customers but about half of revenue, the outage shows up as failures, retries and slow payments, and affected enterprise customers downgrade afterwards.

Include deliberate quality problems for the pipeline to catch: invalid orders (missing customer, non-positive amount, future date), duplicate orders, and unknown payment statuses.

The generator must be re-runnable and able to add one extra day of data as new files later.

## Steps

### Step 1 — Setup and data generation
Check workspace access, create the landing area, and show me a data plan for approval. Then generate the historical data and confirm the story numbers.

### Step 2 — Profile the raw data
Profile every source: volumes, nulls, value distributions and date ranges. Confirm the outage and the bad records are visible, and write a short profile document.

### Step 3 — Medallion pipeline
Build a declarative pipeline in the bundle:
- **Bronze:** incremental ingestion of each source, keeping where each row came from and when it arrived.
- **Silver:** cleaned, typed, de-duplicated data. Customer history is kept as SCD Type 2. Quality rules drop bad rows or warn, and the update fails if too many orders are lost between bronze and silver.
- **Gold:** daily sales and lost revenue, hourly gateway health, per-customer outage impact using each customer's tier *at order time*, and an order-level fact table for the semantic layer.

Validate, deploy and run it.

### Step 4 — Orchestration and incremental load
Add a manually run job that updates the pipeline incrementally and emails on failure. Generate one extra day of data, run the job, and confirm only the new files were processed and the new day reaches gold.

### Step 5 — Semantic layer and dashboard
- Define a governed **metric view** on the gold layer. Measures: revenue, lost revenue, orders, failed orders, payment failure rate, affected enterprise customers. Dimensions: date, hour, region, channel, gateway, tier.
- Build a bundle-managed **dashboard** with:
  - KPIs for the main measures.
  - An hourly failure rate by gateway chart where the outage is obvious.
  - Lost revenue by region and tier.
  - A recovery trend.
  - A top-impacted enterprise customers table.
  - Filters for date, region and gateway.
- Test every query before deploying.

**Done when:** the dashboard is published and clearly shows the outage, its cost, and who was affected.