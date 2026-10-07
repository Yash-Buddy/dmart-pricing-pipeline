# DMart Pricing Pipeline

An end-to-end data engineering project: raw retail product data is ingested, cleaned and modelled in **Databricks** (Bronze → Silver → Gold), served through a **REST API**, and visualised in a **React + D3.js** dashboard.

## Status

| Stage | State |
|-------|-------|
| Profiling | ✅ Done |
| Bronze (ingestion) | ✅ Done, reconciled to source (5,189 rows) |
| Silver (cleaning and modelling) | ✅ Done, reconciled (5,186 rows) |
| Gold (business tables) | ✅ Done, reconciled |
| REST API (Node/Express) | ✅ Built and tested locally (4 endpoints, 5-minute cache) |
| React + D3 dashboard | 🚧 Next |
| Snapshot fallback, deployment | 📋 Planned |

## Architecture

```
DMart.csv ─► pandas parse ─► Bronze ─► Silver ─► Gold ─► REST API ─► React + D3 dashboard
 (Volume)                    raw copy   cleaned   business  Node/Express   charts and filters
                                        + parsed  tables    serves JSON
```

All Databricks code is versioned in this repo through a Databricks Git folder.

## Dataset

Product catalogue of an Indian retail chain, 5,189 products, 9 columns:
`Name, Brand, Price, DiscountedPrice, Category, SubCategory, Quantity, Description, BreadCrumbs`.

It is a product catalogue, not a sales log: there are no dates or transaction rows, so the analysis focuses on pricing, discounts, brands and pack sizes. A few values in `Category` are brand names (for example `Geep`, `Butterfly`); these exist in the source file and are not a parsing error.

## Pipeline layers

| Layer | File | What it does |
|-------|------|--------------|
| Profiling | `01_profiling` | Counts nulls, zero prices, distinct brands and categories before any cleaning |
| Bronze | `02_bronze` (Python notebook) | Reads the original CSV with pandas and writes `bronze_products`, adding `_ingested_at` and `_source_file` |
| Silver | `03_silver.sql` | Builds `silver_products`: drops 3 invalid rows, trims text, parses `Quantity` into `weight_g` / `volume_ml`, adds `discount_pct` |
| Gold | `04_gold.sql` | Builds four dashboard-ready tables (below) |
| Checks | inline | Row-count reconciliation after each layer |

### Silver details
- **Dropped rows (3):** one with no product name, one with a null price, one with a price of 0.
- **`Quantity` parsing:** free text such as `500 gm`, `1.25 kg`, `250 ml`, `1 L`. Converted to grams or millilitres. Values like `Pack of 5` or `Navy Blue` stay NULL because they carry no weight or volume (2,673 rows have a weight, 1,103 have a volume).
- **`discount_pct`:** `(mrp - selling_price) / mrp × 100`. No negative discounts were found.
- **Known issue:** a few products have no category (NULL). Planned fix: label them `Unknown` in Silver so they stay visible in charts.

### Gold tables
| Table | Answers | Rules |
|-------|---------|-------|
| `gold_discount_by_category` | Which categories give the deepest discounts? | Categories with at least 10 products, so tiny categories do not dominate the ranking |
| `gold_top_brands` | Which brands have the most products and what discounts do they give? | Known brands with at least 5 products |
| `gold_price_bands` | How many products fall in each price band, per category? | All products |
| `gold_price_per_100g` | Which food sub-categories are cheapest per 100 g? | Food categories only, packs of 50 g or more, sub-categories with at least 5 products |

## Data quality checks

| Check | Expected | Result |
|-------|----------|--------|
| Bronze rows vs source file | 5,189 | ✅ 5,189 |
| Bronze distinct categories | 29 | ✅ 29 |
| Silver rows (Bronze minus 3 invalid) | 5,186 | ✅ 5,186 |
| Negative discounts in Silver | 0 | ✅ 0 |
| Gold price-band total vs Silver | 5,186 | ✅ 5,186 |
| Gold category total | 5,153 | ✅ 5,153; the other 33 products sit in categories with fewer than 10 products, excluded on purpose (5,186 − 5,153) |

## Data engineering rules followed
1. **Reconcile row counts after every layer.** A mismatch is investigated before moving on.
2. **Keep Bronze raw.** Only metadata columns are added, and the original file is never edited.
3. **Fix problems in the SQL, not in the dashboard.** The dashboard reads only Gold.
4. **Sanity-check the numbers.** If a value cannot be true, trace it to the source rows before trusting it.

## Lessons learned

| Issue | What happened | Resolution |
|-------|---------------|------------|
| Row count mismatch at ingestion | The CSV has 5,189 records, but the upload wizard and then Spark's `read_files` produced 12,434 and 7,249 rows. `Description` contains commas and line breaks inside quoted text (1,033 records), and the reader split records and shifted columns, putting description fragments into the `Category` column. | Caught by the post-load row-count check. Ingested with pandas, which parses quoted CSV correctly; 5,189 rows confirmed. |
| Delta schema mismatch on re-write | Overwriting the old Bronze table with a different column layout failed with `DELTA_METADATA_MISMATCH`. | Dropped the broken table and wrote it again. Delta overwrite replaces data, not schema, unless explicitly allowed. |
| `GROUP BY` position error | `GROUP BY 3` pointed at an aggregate (`COUNT(*)`) in the select list. | Grouped by positions 1 and 2, the non-aggregated columns. |
| Stale table masking a failed write | A failed notebook write left the old, broken table in place, so checks still showed the old numbers. | Always re-run the reconcile query after a write and compare it to the source. |
| Unreliable averages for tiny categories | Categories with 1 to 3 products (for example `Geep`, `Smartwatches`) topped the discount ranking. | Added `HAVING COUNT(*) >= 10` to the discount table. |
| Absurd price per 100 g | Kajal pencils weighing 0.35 g produced about 26,000 per 100 g. Price per weight only makes sense for food. | Restricted the metric to food categories and packs of 50 g or more. |
| Mislabelled sub-category | 32 food products (protein bars, honey, supplements) carry the sub-category "Personal Care" under "Packaged Food". | Inspected the products, found them to be food, and kept them. |
| API pitfalls | A failed Databricks query still returns HTTP 200, and a wrong token returns 401. | The API checks `status.state`, and secrets are read from environment variables only. |

## REST API

A small Node/Express service queries the Gold tables through the Databricks **SQL Statement Execution API** and exposes its own endpoints. It runs locally on port 3000.

How the Databricks API works:
- `POST /api/2.0/sql/statements` with `warehouse_id`, `statement` and `wait_timeout`.
- Short queries return rows inline; otherwise the response carries a `statement_id` that is polled with `GET /api/2.0/sql/statements/{id}`.
- A failed query still returns HTTP 200 with `status.state = FAILED`, so the service checks the state, not only the HTTP status.

Endpoints:

| Endpoint | Returns |
|----------|---------|
| `GET /api/discount-by-category` | Average discount and average MRP per category |
| `GET /api/top-brands` | Top 20 brands by product count, with average discount |
| `GET /api/price-bands` | Product counts per price band and category |
| `GET /api/price-per-100g` | Average price per 100 g by food sub-category |

Details:
- Numeric columns are returned as JSON numbers, not strings.
- Results are cached in memory for 5 minutes to avoid waking the warehouse on every request.
- **Security:** the Databricks token, host and warehouse ID are read from environment variables. They are never committed, and the browser never sees them.
- **Planned:** a JSON snapshot fallback, so the dashboard still works if the warehouse is asleep or the token has expired.

## Dashboard (next)

React + D3.js, reading only the API endpoints above:
- Bar chart: average discount by category
- Ranked list: top brands
- Stacked bars: price bands per category
- Chart: price per 100 g by sub-category
- Filter by category

## Tech stack
Databricks (Delta tables, SQL, Python notebooks, Volumes, SQL Statement Execution API) · pandas · Git · Node.js / Express · React + D3.js _(planned)_

## Repo structure
```
dmart-pricing-pipeline/
├── 01_profiling      # profiling queries
├── 02_bronze         # pandas ingestion notebook
├── 03_silver.sql
├── 04_gold.sql
├── api/              # Node/Express REST API (to be added to this repo)
├── dashboard/        # React + D3 app (planned)
└── README.md
```

## How to run
1. Create a Volume `workspace.default.raw` and upload the original `DMart.csv` (the data file is not stored in this repo).
2. Run `02_bronze`, then `03_silver.sql`, then `04_gold.sql`, checking each layer's row count.
3. Start the API: set `DBX_HOST`, `DBX_WAREHOUSE` and `DBX_TOKEN` as environment variables, then run `node index.js` and open `http://localhost:3000/api/discount-by-category`.
4. Dashboard instructions will be added when built.

## Author
Yash Yadav · [GitHub](https://github.com/Yash-Buddy)
