# DMart Pricing Pipeline

An end-to-end data engineering project: raw retail product data is ingested, cleaned and modelled in **Databricks SQL** (Bronze → Silver → Gold), served through a **REST API**, and visualised in a **React + D3.js** dashboard.

> **Status:** 🚧 In progress. Sections marked _(planned)_ are not built yet.

## Architecture

```
DMart.csv ──► Bronze ──► Silver ──► Gold ──► REST API (Node/Express) ──► React + D3 dashboard
              raw        cleaned    business   serves JSON               charts & filters
                         + modelled tables
```

All SQL runs in Databricks and is versioned in this repo through a Databricks Git folder.

## Dataset

Product catalogue of an Indian retail chain (DMart), about 5,200 products, 9 columns:
`Name, Brand, Price, DiscountedPrice, Category, SubCategory, Quantity, Description, BreadCrumbs`.

Known data issues the Silver layer handles:
- `Quantity` is free text (`500 gm`, `1.25 kg`, `1 L`, `250 ml`, `1 U`) and needs parsing into value and unit
- A few rows have missing `Name` or `Price` values, and some prices are zero
- `Description` contains long marketing text and boilerplate
- Category information is repeated across `Category`, `SubCategory` and `BreadCrumbs`

## Pipeline layers

| Layer | File | Purpose |
|-------|------|---------|
| Profiling | `sql/01_profiling.sql` | Inspect nulls, duplicates, odd values before cleaning |
| Bronze | `sql/02_bronze.sql` | Raw copy of the CSV, unchanged |
| Silver | `sql/03_silver.sql` | Clean types, parse quantity, discount %, dimension tables |
| Gold | `sql/04_gold.sql` | Aggregates for the dashboard (discount by category, top brands, price bands) |
| Checks | `sql/05_checks.sql` | Row-count and key-integrity checks between layers |

## Planned dashboard questions
- Which categories offer the deepest discounts?
- Which brands have the most products and the highest average discount?
- How are prices distributed within each category?
- Which products give the best price per unit (per 100 g / per litre)?

## REST API _(planned)_
A small Node/Express service will query the Gold tables through the Databricks SQL Statement Execution API and expose endpoints such as:

| Endpoint | Returns |
|----------|---------|
| `GET /api/discount-by-category` | Average discount per category |
| `GET /api/top-brands` | Brands ranked by product count |

Credentials are kept in environment variables (`.env` is git-ignored).

## Tech stack
Databricks SQL · Delta tables · Git · Node.js / Express _(planned)_ · React + D3.js _(planned)_

## Repo structure
```
dmart-pricing-pipeline/
├── sql/          # pipeline SQL, one file per layer
├── api/          # REST API (planned)
├── dashboard/    # React + D3 app (planned)
└── README.md
```

## How to run
1. Upload `DMart.csv` to a Databricks Volume (the data file is not stored in this repo).
2. Run the files in `sql/` in order: 01 → 05.
3. API and dashboard instructions will be added when built.

## Author
Yash Yadav · [GitHub](https://github.com/Yash-Buddy)
