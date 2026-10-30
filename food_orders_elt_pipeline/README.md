# Food Orders Batch ELT Pipeline

## Objective
This project implements a batch-based ELT-style workflow with Python, Pandas and MySQL. It loads the original CSV into a raw database first, retrieves the raw records in batches, transforms and validates them, and appends valid records to a separate analytics database.


## Requirements
- Python 3.10+
- MySQL Server running and accessible
- `food_orders.csv` in this folder

## Setup
1. Create and activate a virtual environment (optional but recommended):
   - Windows: `python -m venv .venv` then `.venv\\Scripts\\activate`
   - macOS/Linux: `python3 -m venv .venv` then `source .venv/bin/activate`
2. Install dependencies: `pip install -r requirements.txt`
3. Copy `.env.example` to `.env` and enter your MySQL host, port, username and password. Never commit `.env` to source control.
4. Make sure MySQL Server is running. The pipeline creates `food_orders_raw_db` and `food_orders_analytics_db` automatically.
5. Place `food_orders.csv` beside the Python files, or set `CSV_FILE` in `.env`.

## Run
```bash
python elt_batch_pipeline.py
```

The script prints batch numbers, row counts, transaction errors, and a final execution report. It generates `validation_report.json`, `rejected_records.csv`, and `transformed_valid_records.csv`.

## Data handling decisions
- Extraction reads batches of 100 rows and does not clean or convert source data.
- The raw table stores original CSV values as text and permits NULLs and duplicate rows. `ingestion_id` is a generated database key, not a source field.
- Exact duplicates are identified by a SHA-256 fingerprint of every original source column. A shared fingerprint set catches duplicates across batch boundaries. Repeated Order IDs by themselves are not treated as duplicates.
- Timestamp parsing failures and negative delivery durations are rejected with explicit reasons.
- Financial values that cannot be parsed become missing. Missing financial components are not replaced with zero. Net revenue is NULL unless all four formula inputs are present.
- Delivery time is `(Delivery Date and Time - Order Date and Time)` in minutes.
- Platform net revenue is `Commission Fee + Delivery Fee - Payment Processing Fee - Refunds/Chargebacks`.
- Durations over 180 minutes are not automatically rejected; the threshold is only a possible review flag.

## Database tables
- `food_orders_raw_db.raw_food_orders`: original source values plus `ingestion_id`.
- `food_orders_analytics_db.cleaned_food_orders`: typed source columns plus `Delivery_Time_Minutes` and `Platform_Net_Revenue`.

## Business analysis
Open `business_analysis.sql` in MySQL Workbench or run the queries against `food_orders_analytics_db.cleaned_food_orders`. It answers total/average order value, most common payment method and channel, average delivery time, and total net revenue.

## Important rerun note
The loader appends batches and does not overwrite earlier batches. Therefore, running the pipeline repeatedly against the same databases appends another copy of the source file. For a fresh reproducible run, clear the two destination tables (or use fresh databases) before running again. Do not truncate the raw table if you need to preserve prior ingestion history.

## Sample output format
Actual counts depend on the CSV and database state. The program prints a report with:
- source records extracted and source batches processed
- raw records loaded and raw records retrieved
- valid transformed and rejected records
- transformed records loaded
- successful and failed batch counts
- final validation status

Use the generated `validation_report.json` as the authoritative results file; do not substitute guessed sample numbers.
