# EU Labour Market Data Pipeline

**Tech stack:** Apache Airflow · Snowflake · Python · SQL

An automated data pipeline that extracts monthly unemployment data from the [Eurostat public API](https://ec.europa.eu/eurostat/web/main/data/web-services), transforms it into an analytics-ready format, and loads it into Snowflake — running on a monthly schedule via Apache Airflow.

Built as a portfolio project to demonstrate end-to-end data engineering skills in tools commonly used in production data teams.

---

## Architecture

```
Eurostat REST API
      │
      ▼
┌─────────────┐     ┌──────────────┐     ┌──────────────────┐     ┌────────────────┐
│   Extract   │────▶│  Transform   │────▶│  Load (Upsert)   │────▶│  Quality Check │
│  (Python)   │     │  (Pandas)    │     │  (Snowflake)     │     │  (Snowflake)   │
└─────────────┘     └──────────────┘     └──────────────────┘     └────────────────┘
      │                    │                      │
  Raw JSON            Clean DataFrame       UNEMPLOYMENT_MONTHLY
  via XCom            via XCom              (Snowflake table)
```

**Pipeline steps:**
1. **Extract** — calls Eurostat's JSON-stat API for 8 EU countries/regions
2. **Transform** — parses JSON-stat format into a flat, typed DataFrame
3. **Create table** — idempotently creates the Snowflake target table
4. **Load** — upserts rows using a `MERGE` statement (safe to re-run)
5. **Quality check** — validates row counts, null rates, and value ranges

**Schedule:** `0 6 1 * *` — runs at 06:00 UTC on the 1st of every month

---

## Dataset

- **Source:** [Eurostat une_rt_m](https://ec.europa.eu/eurostat/databrowser/view/une_rt_m/default/table) — Monthly unemployment rates
- **Coverage:** EU27, Germany, France, Poland, Italy, Sweden, Finland, Austria
- **Dimensions:** Total (both sexes), age 15–74, % of active population
- **Frequency:** Monthly, from 2000 onward

---

## Project structure

```
eu-labour-pipeline/
├── dags/
│   └── eurostat_labour_pipeline.py   # Airflow DAG (ELT logic)
├── snowflake/
│   ├── 01_setup.sql                  # One-time Snowflake setup
│   └── 02_analytics.sql              # Analytical queries (rolling avg, YoY, etc.)
├── tests/
│   └── test_pipeline.py              # Unit tests for transform logic
├── requirements.txt
└── README.md
```

---

## Getting started

### 1. Clone and install dependencies

```bash
git clone https://github.com/yourname/eu-labour-pipeline.git
cd eu-labour-pipeline
pip install -r requirements.txt
```

### 2. Set up Airflow locally

```bash
export AIRFLOW_HOME=~/airflow
airflow db init
airflow standalone   # starts webserver + scheduler at localhost:8080
```

Copy the DAG into Airflow's dags folder:
```bash
cp dags/eurostat_labour_pipeline.py ~/airflow/dags/
```

### 3. Set up Snowflake (free trial)

Sign up at [snowflake.com](https://signup.snowflake.com/) — no credit card needed, $400 trial credits included.

Run the setup script in your Snowflake worksheet:
```sql
-- snowflake/01_setup.sql
```

### 4. Configure the Snowflake connection in Airflow

In the Airflow UI → Admin → Connections → Add:

| Field | Value |
|-------|-------|
| Connection ID | `snowflake_default` |
| Connection Type | `Snowflake` |
| Account | your Snowflake account identifier |
| Login | your username |
| Password | your password |
| Database | `EU_LABOUR` |
| Schema | `EUROSTAT` |
| Warehouse | `LABOUR_WH` |

### 5. Trigger the DAG

In the Airflow UI, enable and manually trigger `eurostat_labour_pipeline`.

### 6. Run the analytics queries

Open `snowflake/02_analytics.sql` in your Snowflake worksheet to explore the loaded data.

---

## Running tests

```bash
pip install pytest
pytest tests/ -v
```

---

## Key skills demonstrated

| Skill | Where |
|-------|-------|
| Airflow DAG authoring | `dags/eurostat_labour_pipeline.py` |
| Task dependencies & XCom | Extract → Transform → Load chain |
| Snowflake SQL & MERGE upsert | `load_to_snowflake()` + `01_setup.sql` |
| Window functions (LAG, AVG OVER) | `02_analytics.sql` |
| Data quality checks | `run_data_quality_checks()` task |
| REST API integration | Eurostat JSON-stat parsing |
| Unit testing | `tests/test_pipeline.py` |
| Idempotent pipeline design | MERGE prevents duplicate rows on re-runs |

---

## Author

**Cindy Waweru** — Policy researcher & data analyst  
[linkedin.com/in/cindywaweru](https://linkedin.com/in/cindywaweru) · [medium.com/@cindy.w.waweru](https://medium.com/@cindy.w.waweru)
