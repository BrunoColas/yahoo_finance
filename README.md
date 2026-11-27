# Banking Stock Pipeline (simple): yFinance to Metabase

A trimmed version of the "AI-Driven Banking Stock Analytics Pipeline with DataOps" project.
Removed: email alerts, AI log agent, RAG chatbot, DataOps monitoring DAGs, ML, anomaly and sector DAGs, Spark, MinIO (its Docker images are no longer available; CSV copies now go to the `data/` folder).

```
yFinance --> Airflow (7 DAGs) --> PostgreSQL tables --> Metabase dashboards
                     \--> data/ folder (CSV copy of every stage)
```

## What is inside
| DAG | What it does | Output table |
|---|---|---|
| banking_master_pipeline_dag | Runs the six DAGs below in order, 6:00 AM IST Mon-Fri (00:30 UTC) | - |
| banking_stock_ingestion_dag | Downloads daily prices for 8 NSE bank stocks (last 365 days) | banking_raw_prices |
| banking_indicator_dag | MA20, MA30, RSI, MACD, volatility, volume spike, drawdown | banking_indicators |
| banking_fundamentals_dag | P/E, market cap, beta, EPS | banking_fundamentals |
| banking_feature_engineering_dag | Joins the two and adds trend, momentum, volume flags | banking_features_final |
| banking_scoring_dag | 0-9 score per stock from the latest row | banking_stock_scores |
| banking_dashboard_dag | Latest snapshot and top 10 picks for dashboards | banking_dashboard_latest, banking_dashboard_top_picks |

Each stage also saves a CSV copy under `data/` (raw/, indicators/, fundamentals/, features/, scores/, dashboard/).

## Run it
Needs Docker Desktop (6 GB memory or more) and internet access (for yfinance).

```bash
cp .env.example .env          # already provided; change passwords if you like
chmod -R 777 logs dags plugins data   # needed in Codespaces so Airflow can write
docker compose up --build -d
```
First start takes a few minutes. Then:

1. Open Airflow at http://localhost:8080 (admin / value of AIRFLOW_ADMIN_PASSWORD, default admin).
2. Unpause the 7 DAGs (toggle on the left). DAGs start paused.
3. Open `banking_master_pipeline_dag` and press Trigger. Wait until all six tasks are green.
4. Check the CSV copies in the `data/` folder (for example `ls data/scores`).

The Postgres databases `stock_pipeline_db` (data) and `metabase` (Metabase settings) are created automatically on first start, and the Airflow connection `stock_pipeline_postgres` is set from the environment. No manual setup is needed.

## Connect Metabase
1. Open http://localhost:3000 and finish the first-time setup (create an admin account).
2. Add your data: choose PostgreSQL with
   - Host `postgres`, Port `5432`
   - Database name `stock_pipeline_db`
   - Username `airflow`, Password `airflow` (or your .env values)
3. Browse data, open `banking_dashboard_top_picks`, and build questions, for example:
   - Bar chart: symbol vs final_score (top picks)
   - Table: symbol, close, rsi, macd, pe_ratio, final_score from `banking_dashboard_latest`
   - Line chart: date vs close by symbol from `banking_features_final`
   - Stacked bar of the score parts from `banking_stock_scores`
4. Save the questions to a dashboard.
Tables are fully reloaded on each run, so refresh the dashboard after the pipeline finishes.

## Stop
```bash
docker compose down        # keep data
docker compose down -v     # also delete data volumes
```

## Notes
- Score rules: trend +2, momentum +2, volume spike +1, P/E under 12 = +2 (12 to 20 = +1), beta under 0.5 +1, drawdown below -5% +1. Maximum 9.
- Change the history window with `LOOKBACK_DAYS` in `dags/banking_stock_ingestion_dag.py`. The original used 60 days, which leaves only about 10 rows per stock after the 30-day rolling windows, so this version uses 365.
- Yahoo Finance sometimes rate-limits or returns empty data; re-trigger the ingestion DAG.
- Not tested end to end here: the files were assembled and syntax-checked, but Docker was not run.
