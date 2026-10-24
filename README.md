# BLACK_DRAGONS
# SHASHWAT SINHA , ADITYA KUMAR SINGH , SWATI GUPTA , PIYUSH GARG
ShopSphere E-Commerce Analytics Data Engineering Assignment Pipeline:  
Load raw data Clean &amp; 
fix issues 
Build dimensional model Compute KPIs

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RAW / BRONZE LAYER                           │
│  customers.csv  orders.csv  order_items.csv  payments.csv           │
│  products.csv   returns.csv  suppliers_json.txt                     │
│                  (immutable copies, loaded as-is)                   │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Profile + Cleanse
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       SILVER / CLEANSED LAYER                       │
│  dim_customers   dim_products   dim_suppliers   dim_branches(date)  │
│  fact_orders     fact_order_items   fact_payments   fact_returns     │
│              (validated, deduplicated, standardized)                │
└────────────────────────────┬────────────────────────────────────────┘
                             │ Model + Aggregate
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        GOLD / KPI LAYER                             │
│  KPI 1–8 views/tables (net revenue, CLV, segments, MoM, etc.)      │
└─────────────────────────────────────────────────────────────────────┘
```
---

## 3. Data Model — Star Schema

```
                        ┌─────────────────┐
                        │  dim_date        │
                        │  date_key (PK)   │
                        │  year, month     │
                        │  quarter, week   │
                        └────────┬────────┘
                                 │
          ┌──────────────────────┼───────────────────────┐
          │                      │                        │
┌─────────▼────────┐   ┌─────────▼────────┐   ┌─────────▼────────┐
│  dim_customers   │   │  dim_products    │   │  dim_suppliers   │
│  customer_sk(PK) │   │  product_sk(PK)  │   │  supplier_sk(PK) │
│  customer_id(NK) │   │  product_id(NK)  │   │  supplier_id(NK) │
│  first_name      │   │  product_name    │   │  supplier_name   │
│  last_name       │   │  category        │   │  country         │
│  email           │   │  sub_category    │   │  rating_norm     │
│  segment         │   │  unit_price      │   │  is_preferred    │
│  is_active       │   │  cost_price      │   │  contract_start  │
│  signup_date     │   │  supplier_sk(FK) │   └──────────────────┘
│  scd_start_date  │   │  is_active       │
│  scd_end_date    │   │  launch_date     │
│  is_current      │   └──────────────────┘
└─────────┬────────┘
          │
          │        ┌─────────────────────────────────────────┐
          │        │           fact_order_items              │
          └────────►  order_item_sk (PK, surrogate)          │
                   │  order_item_id (NK)                     │
                   │  order_id (FK → fact_orders)            │
                   │  product_sk (FK → dim_products)         │
                   │  customer_sk (FK → dim_customers)       │
                   │  date_key (FK → dim_date)               │
                   │  quantity                               │
                   │  unit_price                             │
                   │  discount_pct                           │
                   │  line_total (recomputed)                │
                   └──────────────┬──────────────────────────┘
                                  │
                    ┌─────────────┴──────────────┐
          ┌─────────▼────────┐       ┌───────────▼──────────┐
          │  fact_payments   │       │  fact_returns        │
          │  payment_sk(PK)  │       │  return_sk (PK)      │
          │  payment_id(NK)  │       │  return_id (NK)      │
          │  order_id (FK)   │       │  order_item_id (FK)  │
          │  date_key (FK)   │       │  date_key (FK)       │
          │  amount          │       │  refund_amount       │
          │  payment_status  │       │  return_reason       │
          │  payment_method  │       │  return_status       │
          └──────────────────┘       └──────────────────────┘
```

### SCD Type 2 on dim_customers
Customer `segment` and `address` can change meaningfully over time.

```
customer_sk | customer_id | segment  | scd_start  | scd_end    | is_current
------------|-------------|----------|------------|------------|----------
1001        | C0015       | Bronze   | 2021-01-02 | 2023-06-14 | false
1002        | C0015       | Silver   | 2023-06-15 | 9999-12-31 | true
```

- New record inserted on attribute change; previous row's `scd_end_date` closed.
- `fact_order_items` joins to `customer_sk` valid at `order_date` (point-in-time join).

---

## 4. Pipeline Code Structure

```
pppp/
├── data/
│   ├── raw/                  # Extracted zip contents (immutable)
│   └── silver/               # Cleansed Parquet files
│       ├── dim_customers.parquet
│       ├── dim_products.parquet
│       ├── dim_suppliers.parquet
│       ├── fact_order_items.parquet
│       ├── fact_payments.parquet
│       ├── fact_returns.parquet
│       └── rejects/          # Quarantined rows with reason codes
├── notebooks/
│   ├── 01_profiling.ipynb    # EDA on all 7 raw files
│   ├── 02_cleansing.ipynb    # Apply cleansing rules → silver layer
│   ├── 03_modeling.ipynb     # Build star schema in DuckDB
│   └── 04_kpis.ipynb         # Compute all 8 KPIs
├── sql/
│   ├── ddl/                  # CREATE TABLE statements
│   └── kpis/                 # One .sql file per KPI
├── DESIGN.md                 # This file
└── README.md
```

---

---
