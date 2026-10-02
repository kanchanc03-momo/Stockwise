# Stockwise

A retail sales forecasting and stock planning application built with React, FastAPI, PostgreSQL, and PySpark. This is a working pilot implementation, not a claim of production certification.

## Run with Docker

Install Docker Desktop (Linux containers), copy `.env.example` to `.env`, and replace both secrets. Use an alphanumeric database password to avoid URL encoding issues.

```powershell
Copy-Item .env.example .env
docker compose up --build -d
```

Open http://localhost:8080. Create an account, upload `sample-data/sales.csv`, and start a forecast from Sales data. The worker trains five models; expect several minutes depending on the machine. Add inventory using `sample-data/inventory.csv` as a reference, then create a plan from Forecasts, edit quantities, save, and export.

On Windows, after Docker Desktop is running, `powershell -ExecutionPolicy Bypass -File .\Start-Stockwise.ps1` checks the engine and starts all services. This deployment uses port 8080; the separate development frontend uses port 5173.

Allow at least 6 GB RAM for Docker. Ports for the database and API are not exposed; Nginx serves the UI and proxies `/api`. Volumes persist the database, sales files, and models.

## Local development

Backend requires Python 3.11 and Java 17 (the Docker image provides both). The host's Python 3.14 is not the supported Spark runtime.

```powershell
py -3.11 -m venv .venv
.venv\Scripts\python -m pip install -r backend/requirements.txt
$env:JWT_SECRET = 'a-long-development-secret-of-at-least-32-characters'
$env:DATA_DIR = "$PWD/data"
cd backend
..\.venv\Scripts\python -m uvicorn app.main:app --reload
```

In another terminal with the same environment and backend directory, run `python -m app.worker`. Without DATABASE_URL, development uses SQLite. Run a single worker with SQLite; PostgreSQL is required for multiple workers. Start the frontend with `npm.cmd install` and `npm.cmd run dev` from `frontend`. API documentation is at http://localhost:8000/docs.

## CSV contract

Required header: `date,product_id,quantity,price`. Dates use YYYY-MM-DD; units are integers; price is nonnegative. IDs must be consistent between sales and inventory. Optional extra columns are accepted. UTF-8 encoding; up to 50 MB and 2,000 products in this pilot. Each row is a transaction; repeated rows are treated as separate transactions because no transaction ID is supplied. Prices have no currency assumption and forecasts are in units.

Negative quantities are returns. Forecast demand is **gross positive units sold**, with returns reported separately. The newest calendar week is excluded conservatively because the upload cannot prove that its week is complete. Forecasts concern the week after the last completed calendar week in the file, not necessarily the current date. Fresh data is required for current planning.

## Forecast methodology

Weekly product series are zero-filled from each product's first observation. Features: prior 1/2/4-week units, prior four-week mean, month, seasonal sine/cosine, and one-hot product ID. No target-week quantities or prices enter features. Minimum 20 complete weeks; recently introduced products without four weeks of lags are excluded and counted.

Chronological 60/20/20 split by common calendar weeks. All five regressors use the same preprocessing fitted on training. Linear Regression, Decision Tree, Random Forest, Gradient Boosted Trees, and Factorization Machines compete with the last-week baseline. Lowest validation MAE wins, including the baseline if it performs better. The selected model is refitted on training plus validation and evaluated once on untouched test weeks; these are rolling one-week-ahead predictions using observed prior weeks. It is then refitted on all complete history for the next forecast and saved. Fixed seed 42 where supported. Predictions are clipped to zero consistently for metrics and planning. Reported training/prediction timings exclude shared preprocessing. They are practical single-run timings, not controlled benchmarks.

Spark runs local[2] in the worker, demonstrating local parallel processing. Multi-machine scaling needs a Spark cluster and shared storage; this deployment does not claim distributed cluster benchmarks. Size-versus-time charts show actual runs and depend on history and product mix, not only row count.

Restocking uses `ceil(forecast * (lead_days + 7) / 7 + safety_stock - current_stock)`, floored at zero. This assumes uniform daily demand and a weekly review cycle. Missing inventory produces a blank suggestion. Plans snapshot the stock at creation. Recreate a plan after changing inventory. CSV exports include saved edits only.

## Validation

```powershell
cd backend
python -m pytest tests -q
cd ../frontend
npm.cmd run build
```

See `RELEASE.md` for observed checks, deployment requirements, and remaining work before charging customers.

Real UCI benchmark ingestion and attribution are documented in `UCI_DATA.md`. The website's first real-data run uses a clearly labelled 100-product cohort; the original workbook and full cleaned transaction CSV are retained locally.

## Password recovery

The sign-in page includes Forgot password and a reset form. Links expire after 30 minutes, work once, and revoke previous sessions after the password changes. Configure PUBLIC_APP_URL and SMTP_HOST, SMTP_PORT, SMTP_FROM, SMTP_USER, SMTP_PASSWORD in `.env`, then recreate the API container to enable email delivery over STARTTLS. Without SMTP, the screen explains that administrator assistance is required.

For a local owner whose identity has been verified, an operator can run `docker compose exec api python -m app.recovery --store Designx --output /data/recovery/designx-link.txt`. The private file contains the reset URL; give it only to that owner. Generating a link leaves the existing password unchanged. The owner enters the new password on the reset page.
# Scaleable-
