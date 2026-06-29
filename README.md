# Databricks Medallion Pipeline with dbt

A bronze → silver → gold data pipeline built with **dbt-core** on **Databricks SQL Warehouse**, demonstrating layered transformation, data quality testing, and change-tracking via snapshots.

## What this is

This project takes raw retail sales data (sales, products, customers, stores, returns) through a medallion architecture:

- **Bronze** — raw source data landed as-is, with light typing and dbt sources defined for lineage tracking
- **Silver** — cleaned, joined, and enriched business-level tables (e.g. sales joined with product category and customer attributes)
- **Gold** — analytics-ready aggregates and dimensional models, including a **slowly changing dimension (SCD Type 2)** snapshot to track historical changes over time

Each layer is materialized into its own schema in the warehouse (`bronze`, `silver`, `gold`) via a custom `generate_schema_name` macro, instead of dbt's default schema-concatenation behavior — keeping the catalog clean and matching how a real analytics team would organize a lakehouse.

## Architecture

```
Databricks Unity Catalog (dbt_tutorial_dev)
│
├── source schema          (raw landed tables)
│
├── bronze schema           ← models/bronze/*.sql
│     bronze_sales, bronze_product, bronze_customer,
│     bronze_store, bronze_date, bronze_returns
│
├── silver schema           ← models/silver/*.sql
│     silver_salesinfo (sales enriched with product + customer attributes)
│
└── gold schema             ← models/gold/*.sql + snapshots/
      source_gold_items
      gold_items snapshot (SCD Type 2, tracks historical state via dbt snapshots)
```

## Highlights / what's worth a closer look

- **Custom `generate_schema_name` macro** (`macros/generate_schema.sql`) — overrides dbt's default behavior so each medallion layer builds into a schema named exactly after itself (`bronze`, `silver`, `gold`), rather than dbt's default `<target_schema>_<custom_schema>` pattern.
- **Custom generic test** (`tests/generic/generic_non_negative.sql`) — a reusable dbt test asserting that financial columns (e.g. `gross_amount`) never go negative, applied declaratively in YAML across multiple models.
- **Singular test** (`tests/non_negative_test.sql`) — a one-off business rule check that both `gross_amount` and `net_amount` aren't simultaneously negative on the same sales row.
- **SCD Type 2 snapshot** (`snapshots/gold_items.yml`) — uses dbt's timestamp-based snapshot strategy to capture historical changes on the gold item dimension, including `dbt_valid_to_current` for clean "current record" filtering.
- **Source-based lineage** — all bronze models reference raw tables exclusively through `{{ source(...) }}`, defined centrally in `models/source/sources.yml`, so `dbt docs generate` produces full upstream lineage.
- **Data quality test suite** — `unique`, `not_null`, `accepted_values`, and the custom non-negative tests, covering primary keys, categorical integrity, and business rules.

## Tech stack

| Layer | Tool |
|---|---|
| Transformation | dbt-core 1.11 |
| Warehouse | Databricks SQL Warehouse (Serverless) |
| Adapter | dbt-databricks |
| Package management | [uv](https://github.com/astral-sh/uv) |
| Language | SQL + Jinja |

## Project structure

```
dbt_project/
├── models/
│   ├── source/        sources.yml (raw table definitions)
│   ├── bronze/         raw landed models + properties.yml (tests)
│   ├── silver/         cleaned/joined business models
│   └── gold/            analytics-ready models
├── snapshots/         SCD Type 2 snapshot definitions
├── macros/             generate_schema_name override, reusable SQL macros
├── tests/
│   ├── generic/        reusable parametrized test (generic_non_negative)
│   └── non_negative_test.sql   singular business-rule test
├── seeds/              static reference/lookup CSV data
└── dbt_project.yml
```

## Running this project

```bash
# clone and set up environment
git clone https://github.com/Tushar-me24/<repo-name>.git
cd <repo-name>/dbt_project
uv sync
.venv\Scripts\activate   # or: source .venv/bin/activate on macOS/Linux

# configure your own profiles.yml with Databricks credentials
# (see profiles.yml.example — never commit real credentials)

dbt deps
dbt debug      # verify the Databricks connection
dbt seed       # load reference data
dbt run        # build bronze -> silver -> gold
dbt snapshot   # build SCD snapshot
dbt test       # run the full test suite
```

## Notes

- `profiles.yml` is intentionally excluded from version control (`.gitignore`) since it holds warehouse credentials. A `profiles.yml.example` is provided as a template.
- Schema/catalog names (`dbt_tutorial_dev`, etc.) are specific to the development sandbox this was built against and would be parameterized via dbt `target`/`env_var()` in a production setup.
