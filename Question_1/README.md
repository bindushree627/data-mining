# Annapurna Stores — Data Mining & Data Warehouse Lab

## Stack
- MinIO: object store
- PostgreSQL: relational master data
- DuckDB: analytical query engine
- Python + pandas/pyarrow/boto3: ingestion

## VS Code setup

1. Extract the exam ZIP so you have:
   `data_2/data/sales/`
   `data_2/data/masters.sql`
   `data_2/data/finance_monthly.csv`
   `data_2/data/billing_notes.md`

2. Copy the exam `masters.sql` into this project folder, replacing the placeholder.

3. Open this project folder in VS Code.

4. Start infrastructure:
   `docker compose up -d`

5. Create a Python environment:
   Windows PowerShell:
   `py -m venv .venv`
   `.venv\Scripts\Activate.ps1`
   `pip install -r requirements.txt`

6. Run ingestion:
   `python scripts/ingest.py --sales-dir ..\data_2\data\sales`

The script is idempotent at line level. Re-sends are not treated as replacement files.
The key is `(bill_no,line_no)`, and conflicting duplicate keys cause a hard failure.

## Part A evidence

The layout is:

`sales/year=YYYY/month=MM/store=Sxx/SALES_Sxx_YYYYMMDD.parquet`

For one store-month query, only that store/month partition is eligible.
For example S03 October 2024:
`sales/year=2024/month=10/store=S03/`

There are 31 daily source files in that partition. A single-folder design has all 4457 input files eligible.

To record actual curated bytes:
PowerShell:
`(Get-ChildItem -Recurse data\curated\sales\year=2024\month=10\store=S03\*.parquet | Measure-Object Length -Sum).Sum`

## Part B evidence

Run:
`python scripts/validate.py --sales-dir ..\data_2\data\sales`

Run the same command three times. The row count and checksum must remain identical.

For this supplied dataset, the expected canonical result is:
- raw rows: 1,137,585
- deduplicated rows: 1,120,924
- checksum (canonical key/content hash): e0387aa5e8bb331d2f0fea343b6f839a8db500681e52b0c40abf22298a63ce9a

## Part C

Do not join products on product_code alone. The vendor says codes were reissued.
Use product_code + business_date against valid_from/valid_to, resolving to product_sk.
Do not count TAX or TENDER as revenue.
Keep SALE, RETURN, DISCOUNT and VOID so cancellation nets to zero.

## Part D

Historical price lookup is a temporal join against price_revisions:
effective_from <= report_end
AND effective_to >= report_start

Use the same SQL with only the reporting period changed.

## Part E

DuckDB can read Parquet and PostgreSQL in the same query.
Run:
`duckdb`

Then:
`.read sql/03_cross_system.sql`

For evidence, run EXPLAIN ANALYZE on the SELECT. In the plan, look for:
- READ_PARQUET / PARQUET_SCAN: object-store side
- POSTGRES_SCAN / PostgreSQL scan: PostgreSQL side
- HASH_JOIN / FILTER / PROJECTION / AGGREGATE: DuckDB execution

## Part F

Run `sql/05_reconciliation.sql` and compare to `data_2/data/finance_monthly.csv`.

Known expected pipeline reconciliation from the supplied files:
2024-01  38446071.33  exact
2024-02  34887085.55  exact
2024-03  41971649.09  vs 42457899.09  diff -486250.00
2024-04  37958457.37  exact
2024-05  41764716.40  exact
2024-06  38987082.82  exact
2024-07  40295160.11  vs 40527291.81  diff -232131.70
2024-08  45252181.75  exact
2024-09  44615037.46  exact
2024-10  56359195.92  exact
2024-11  51583838.47  exact
2024-12  50745259.48  vs 50745209.00  diff +50.48

July is explained by the documented S07 three-day missing export.
March and December require evidence from the supplied data/finance target before assigning the final cause; do not label them a pipeline bug without that evidence.

## Important examiner trap

Never do:
`SUM(qty * unit_price)` over every line type.

TAX and TENDER are not revenue. TENDER is the bill total and will approximately double-count sales.
