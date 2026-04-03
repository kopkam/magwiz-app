# MAGWIZ — Warehouse Inventory & Logistics Analytics

A multi-page Streamlit dashboard for warehouse management, built on a local SQLite data warehouse fed by Excel files. Designed to give warehouse managers, inventory planners, and logistics coordinators a single place to monitor stock, track supplier/customer performance, and classify products by business value.

![MAGWIZ Logo](MAGWIZ.png)

---

## What it does

MAGWIZ ingests operational Excel exports, loads them into a structured SQLite database, and exposes 7 analytical views:

| Page | Purpose |
|---|---|
| Data Exploration | Sync Excel files to the DB; browse and validate table contents |
| Warehouse Stock | Current stock levels per product/warehouse with trend lines |
| Stock Shortages | Products below safety stock threshold with missing quantity |
| ABC Analysis | Pareto classification of products by sales volume (A/B/C) |
| Order Timeliness | On-time rate for supplier deliveries and customer shipments |
| Order Fulfillment Time | Average lead times by supplier and customer |
| Warehouse Fill Levels | Volumetric capacity utilisation per warehouse |

---

## Data Architecture

The app uses a **star schema** in SQLite with 3 dimension tables and 6 fact/transaction tables:

```
Dimensions:          Fact Tables:
  Products    ──┐      WarehouseStock (daily snapshots)
  Suppliers   ──┼──→   Deliveries + DeliveryDetails
  Customers   ──┤      Orders + OrderDetails
  Warehouses  ──┘
```

**Data flow:**

```
Excel files (data/)
      │
      ▼
Page 1: Data Exploration (validate + sync)
      │
      ▼
SQLite (db_inventory.db)
      │
      ├──→ Page 2: Warehouse Stock
      ├──→ Page 3: Stock Shortages
      ├──→ Page 4: ABC Analysis
      ├──→ Page 5: Order Timeliness
      ├──→ Page 6: Order Fulfillment Time
      └──→ Page 7: Warehouse Fill Levels
```

### Source files expected in `data/`

| File | Description |
|---|---|
| `products.xlsx` | Product master: name, safety stock, physical dimensions |
| `suppliers.xlsx` | Supplier master |
| `customers.xlsx` | Customer master |
| `warehouses.xlsx` | Warehouse master with volumetric capacity |
| `warehouse_stock.xlsx` | Daily stock snapshots per product/warehouse |
| `orders.xlsx` | Customer order headers with dates and shipping status |
| `order_details.xlsx` | Order line items (product + quantity) |
| `deliveries.xlsx` | Supplier delivery headers with dates and delivery status |
| `delivery_details.xlsx` | Delivery line items (product + quantity) |

---

## Key Metrics & Calculations

| Metric | Formula |
|---|---|
| Missing Quantity | `safety_stock − quantity_available` |
| ABC Category | Cumulative % of sales volume → A (0–20%), B (20–50%), C (50–100%) |
| On-Time Rate | `COUNT(status = 'Completed on time') / total × 100` |
| Lead Time | `julianday(delivery_date) − julianday(order_date)` (days) |
| Warehouse Fill % | `SUM(unit_volume × quantity_available) / warehouse_capacity × 100` |

Gauge thresholds: green < 50%, orange 50–75/90%, red > 75/90% (depending on page).

---

## Tech Stack

- **Python 3** — core language
- **Streamlit** — web UI and multi-page navigation (`st_pages`)
- **SQLite 3** — embedded relational database
- **Pandas** — data transformation and Excel I/O
- **Plotly Express / Graph Objects** — interactive charts (line, bar, gauge)
- **openpyxl** — Excel read/write for data sync and downloads

---

## Setup & Running

```bash
# 1. Clone the repo
git clone https://github.com/kopkam/magwiz-app.git
cd magwiz-app

# 2. Install dependencies
pip install streamlit pandas plotly openpyxl st-pages

# 3. Initialise the database (run once)
jupyter nbconvert --to notebook --execute db_schema.ipynb

# 4. Launch the app
streamlit run main.py
```

Open `http://localhost:8501` in your browser.

> On first run, go to **Data Exploration** (Page 1) and click "Update data" for each table to load the Excel files into the database.

---

## Sample Data

Products data sourced from the Kaggle dataset:
[250k Medicines Usage, Side Effects and Substitutes](https://www.kaggle.com/datasets/shudhanshusingh/250k-medicines-usage-side-effects-and-substitutes/data)

---

## Project Structure

```
magwiz-app/
├── main.py                       # App entry point & navigation
├── db_schema.ipynb               # DB initialisation notebook
├── db_inventory.db               # SQLite database (generated)
├── MAGWIZ.png                    # App logo
├── data/                         # Source Excel files
│   ├── products.xlsx
│   ├── suppliers.xlsx
│   ├── customers.xlsx
│   ├── warehouses.xlsx
│   ├── warehouse_stock.xlsx
│   ├── orders.xlsx
│   ├── order_details.xlsx
│   ├── deliveries.xlsx
│   └── delivery_details.xlsx
└── pages/
    ├── 1_data_exploration.py
    ├── 2_warehouse_stock.py
    ├── 3_stock_shortages.py
    ├── 4_abc_analysis.py
    ├── 5_order_timeliness.py
    ├── 6_order_fulfillment_time.py
    └── 7_warehouse_fill_levels.py
```

---

## Potential Improvements (Data Engineering Perspective)

- **Cost-weighted ABC Analysis** — current ABC uses sales quantity; weighting by unit cost/revenue gives a more accurate Pareto classification
- **Inventory transaction log** — current model stores only daily snapshots; a movements table (receipts, shipments, adjustments) would enable shrinkage tracking and delta analysis
- **Automated ETL pipeline** — replace manual Excel sync with a scheduled pipeline (e.g. Airflow, Prefect, or a simple cron job calling the sync logic)
- **Slowly Changing Dimensions** — safety stock thresholds and warehouse capacities change over time; versioning them enables historical KPI accuracy
- **Referential integrity validation** — add data quality checks before loading (null primary keys, orphan foreign keys, type mismatches)
- **Delay root-cause tracking** — current status is binary (on time / delayed); adding a reason code field enables root cause analysis
- **Reorder point automation** — derive safety stock dynamically from average lead time × average daily demand instead of static values
