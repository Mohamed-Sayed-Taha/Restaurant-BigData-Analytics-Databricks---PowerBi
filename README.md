# 🍽️ Restaurant Analytics — End-to-End Big Data Pipeline

![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD8?style=for-the-badge&logo=delta&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![Fivetran](https://img.shields.io/badge/Fivetran-0073E6?style=for-the-badge&logo=fivetran&logoColor=white)
![Google Drive](https://img.shields.io/badge/Google%20Drive-4285F4?style=for-the-badge&logo=googledrive&logoColor=white)

> A production-style **Big Data analytics pipeline** for a multi-branch Egyptian restaurant chain — built on **Databricks**, modeled with **Delta Lake (Star Schema)**, and visualized in **Power BI**. The dataset contains over **11 million rows** of synthetic order data. The primary goal is hands-on mastery of **Databricks**, Spark SQL, and enterprise-grade data modeling — not just dashboard building.

---

## 📌 Table of Contents

- [Project Overview](#-project-overview)
- [Architecture & Data Flow](#-architecture--data-flow)
- [Data Ingestion](#-data-ingestion)
- [Data Modeling — Star Schema](#-data-modeling--star-schema)
- [SQL — Dimension Tables](#-sql--dimension-tables)
- [SQL — Fact Table](#-sql--fact-table)
- [DAX Measures (Power BI)](#-dax-measures-power-bi)
- [Power BI Dashboard Pages](#-power-bi-dashboard-pages)
- [Key Insights](#-key-insights)
- [Repository Structure](#-repository-structure)
- [Author](#-author)

---

## 📊 Project Overview

This project simulates a real-world analytics engineering workflow applied to restaurant operations. The pipeline covers everything from raw file ingestion to interactive executive dashboards.

| Layer | Tool Used |
|---|---|
| ☁️ Data Storage | Google Drive |
| 🔄 Data Ingestion | Fivetran → Databricks |
| ⚡ Data Processing | Apache Spark SQL (Databricks) |
| 🗄️ Storage Format | Delta Lake |
| 🧩 Data Modeling | Star Schema (SQL) |
| 📊 Visualization | Power BI (5 Dashboard Pages) |

**Dataset at a Glance:**

| Metric | Value |
|---|---|
| Total Rows | 11.110 Million |
| Total Revenue | $2.898 Billion |
| Avg Order Value | $261 |
| Avg Rating | 3.7 |
| Number of Customers | 200K |
| Repeat Customer Rate | 100% |
| Branches | 6 Egyptian cities |
| Menu Items | 15 distinct items |
| Years Covered | 2020 – 2025 |

> ⚠️ **The data is synthetic / fake.** This project is focused on practicing the full Databricks pipeline — ingestion, transformation, modeling, and visualization — at scale.

---

## 🏗️ Architecture & Data Flow

```
Google Drive (CSV × 7 + JSON × 2)
        │
        ▼
   Fivetran (Auto-sync connector)
        │
        ▼
  Databricks Workspace
        │
   ┌────┴────┐
   │         │
resturant_csv  resturant_json
   │         │
   └────┬────┘
        │  UNION ALL
        ▼
  restaurant_data  (Delta Lake — unified raw table)
        │
        ▼
  ┌─────────────────────────────────────────────┐
  │           Star Schema (Delta Lake)          │
  │                                             │
  │  Dim_branch      Dim_category               │
  │  Dim_item        Dim_payment_method         │
  │  Dim_order_type  Dim_date                   │
  │                    ▼                        │
  │              Fact_orders                    │
  └─────────────────────────────────────────────┘
        │
        ▼
  Power BI (DirectQuery / Import)
        │
        ▼
  5 Dashboard Pages
```

---

## 📥 Data Ingestion

### The Problem
The original approach was loading files **manually from Google Drive URLs directly into Databricks** — a process that took **hours** due to API rate limits and connection instability.

### The Solution: Fivetran
Connected Google Drive to **Fivetran**, which automatically synced all source files into Databricks as registered tables — cutting ingestion time from hours to minutes and making the pipeline **repeatable and production-ready**.

### Step 1 — Merge all CSV files into one table

```sql
-- Merge all CSV files
CREATE TABLE IF NOT EXISTS restaurant_csv
USING DELTA
as
SELECT * from workspace.google_drive.restaurant_1
UNION ALL
SELECT * from workspace.google_drive.restaurant_2
UNION ALL
SELECT * from workspace.google_drive.restaurant_3
UNION ALL
SELECT * from workspace.google_drive.restaurant_4
UNION ALL
SELECT * from workspace.google_drive.restaurant_5
UNION ALL
SELECT * from workspace.google_drive.restaurant_6
UNION ALL
SELECT * from workspace.google_drive.restaurant_7;
```

### Step 2 — Merge all JSON files into one table

```sql
-- Merge all JSON files
CREATE TABLE IF NOT EXISTS restaurant_json
USING DELTA
as
SELECT * from workspace.google_drive.restaurant_json_1
UNION ALL
SELECT * from workspace.google_drive.restaurant_json_2
```

### Step 3 — Merge CSV + JSON into unified raw table

```sql
-- Merge CSV and JSON files
CREATE TABLE IF NOT EXISTS restaurant_data
USING DELTA
as
SELECT * from resturant_csv
UNION ALL
SELECT * from resturant_json
```

> The unified `restaurant_data` table contains **11.110M rows** and is the single source of truth for all downstream modeling.

---

## 🗄️ Data Modeling — Star Schema

The data model follows a **Star Schema** design — one central Fact table surrounded by 6 Dimension tables, all stored as **Delta Lake** tables in Databricks and connected in Power BI.

### Schema Diagram

<img width="1010" height="752" alt="Schema" src="https://github.com/user-attachments/assets/3e6a7763-d4e2-4bff-89bb-0860547a19cb" />


| Table | Type | Key Column |
|---|---|---|
| `Fact_orders` | Fact | — |
| `Dim_branch` | Dimension | `branch_key` |
| `Dim_category` | Dimension | `category_key` |
| `Dim_item` | Dimension | `item_key` |
| `Dim_payment_method` | Dimension | `payment_method_key` |
| `Dim_order_type` | Dimension | `order_type_key` |
| `Dim_date` | Dimension | `date_key` |

All dimension tables use `ROW_NUMBER() OVER (ORDER BY ...)` to generate **surrogate keys**. All relationships are **one-to-many** from each dimension to `Fact_orders`.

---

## 🧩 SQL — Dimension Tables

### Dim_branch

```sql
-- Create Dim_branch
CREATE OR REPLACE TABLE Dim_branch
USING DELTA AS
WITH distinct_branches AS (
    SELECT DISTINCT branch
    FROM workspace.default.resturant_data
    WHERE branch IS NOT NULL
)

SELECT
    branch,
    ROW_NUMBER() OVER (ORDER BY branch) AS branch_key
FROM distinct_branches;
```

### Dim_category

```sql
-- Create Dim_category
CREATE OR REPLACE TABLE Dim_category
USING DELTA AS
WITH distinct_category AS (
    SELECT DISTINCT category
    FROM workspace.default.resturant_data
    WHERE category IS NOT NULL
)

SELECT
    category,
    ROW_NUMBER() OVER (ORDER BY category) AS category_key
FROM distinct_category;
```

### Dim_item

```sql
-- Create Dim_item
CREATE OR REPLACE TABLE Dim_item
USING DELTA AS
WITH distinct_item_name AS (
    SELECT DISTINCT item_name
    FROM workspace.default.resturant_data
    WHERE item_name IS NOT NULL
)

SELECT
    item_name,
    ROW_NUMBER() OVER (ORDER BY item_name) AS item_key
FROM distinct_item_name;
```

### Dim_payment_method

```sql
-- Create Dim_payment_method
CREATE OR REPLACE TABLE Dim_payment_method
USING DELTA AS
WITH distinct_payment_method AS (
    SELECT DISTINCT payment_method
    FROM workspace.default.resturant_data
    WHERE payment_method IS NOT NULL
)

SELECT
    payment_method,
    ROW_NUMBER() OVER (ORDER BY payment_method) AS payment_method_key
FROM distinct_payment_method;
```

### Dim_order_type

```sql
-- Create Dim_order_type
CREATE OR REPLACE TABLE Dim_order_type
USING DELTA AS
WITH distinct_order_type AS (
    SELECT DISTINCT order_type
    FROM workspace.default.resturant_data
    WHERE order_type IS NOT NULL
)

SELECT
    order_type,
    ROW_NUMBER() OVER (ORDER BY order_type) AS order_type_key
FROM distinct_order_type;
```

### Dim_date

```sql
-- Create Dim_date
CREATE OR REPLACE TABLE Dim_date
USING DELTA AS
WITH date_range AS (
    SELECT
        MIN(order_date) AS start_date,
        MAX(order_date) AS end_date
    FROM resturant_data
),
dates AS (
    SELECT explode(sequence(start_date, end_date, interval 1 day)) AS date
    FROM date_range
)

SELECT
    -- PRIMARY KEY
    CAST(date_format(date, 'yyyyMMdd') AS INT) AS date_key,
    CAST(date AS DATE)                          AS date,
    CAST(year(date) AS INT)                     AS year,
    CAST(quarter(date) AS INT)                  AS quarter,
    CAST(month(date) AS INT)                    AS month_number,
    CAST(date_format(date, 'MMMM') AS STRING)   AS month_name,
    CAST(day(date) AS INT)                      AS day_number,
    CAST(date_format(date, 'EEEE') AS STRING)   AS day_name,
    CAST(dayofweek(date) AS INT)                AS day_of_week
FROM dates;
```

> `Dim_date` is generated entirely from the **min/max order date range** using Spark's `sequence()` and `explode()` functions — no external date table needed.

---

## 📋 SQL — Fact Table

The Fact table joins all dimensions to the raw data and enriches it with a derived `time_slot` field based on the order hour.

```sql
-- Create Fact_orders
CREATE OR REPLACE TABLE Fact_orders
USING DELTA AS
SELECT
    r.order_id,
    r.order_date,
    d.date_key,
    r.hour,
    CASE
        WHEN CAST(hour AS INT) BETWEEN 6  AND 10 THEN 'breakfast'
        WHEN CAST(hour AS INT) BETWEEN 11 AND 14 THEN 'lunch'
        WHEN CAST(hour AS INT) BETWEEN 15 AND 17 THEN 'afternoon'
        WHEN CAST(hour AS INT) BETWEEN 18 AND 22 THEN 'dinner'
        ELSE 'late night'
    END AS time_slot,
    c.category_key,
    i.item_key,
    r.price,
    r.quantity,
    r.discount,
    r.total_amount,
    b.branch_key,
    p.payment_method_key,
    ot.order_type_key,
    r.customer_id,
    r.rating,
    r.is_weekend
FROM workspace.default.restaurant_data AS r
LEFT JOIN dim_branch          AS b  ON r.branch         = b.branch
LEFT JOIN dim_category        AS c  ON r.category       = c.category
LEFT JOIN dim_item            AS i  ON r.item_name      = i.item_name
LEFT JOIN dim_payment_method  AS p  ON r.payment_method = p.payment_method
LEFT JOIN dim_order_type      AS ot ON r.order_type     = ot.order_type
LEFT JOIN dim_date            AS d  ON r.order_date     = d.date;
```

**Fact_orders Columns:**

| Column | Description |
|---|---|
| `order_id` | Unique order identifier |
| `order_date` | Date of the order |
| `date_key` | FK → Dim_date |
| `hour` | Hour of the order (raw) |
| `time_slot` | Derived: breakfast / lunch / afternoon / dinner / late night |
| `category_key` | FK → Dim_category |
| `item_key` | FK → Dim_item |
| `price` | Unit price |
| `quantity` | Items ordered |
| `discount` | Discount applied |
| `total_amount` | Final order amount |
| `branch_key` | FK → Dim_branch |
| `payment_method_key` | FK → Dim_payment_method |
| `order_type_key` | FK → Dim_order_type |
| `customer_id` | Customer identifier |
| `rating` | Order rating (1–5) |
| `is_weekend` | Weekend flag |

---

## 📐 DAX Measures (Power BI)

All measures are organized in a dedicated `_Measures` table in Power BI.

```dax
-- ─────────────────────────────────────────
-- REVENUE MEASURES
-- ─────────────────────────────────────────

Total Revenue =
SUM(Fact_orders[total_amount])

Total Rev before discount =
SUMX(
    Fact_orders,
    Fact_orders[price] * Fact_orders[quantity]
)

Avg Order Value =
DIVIDE([Total Revenue], [Total Num of Orders], 0)

AVG price =
AVERAGE(Fact_orders[price])

Total Discount =
SUMX(
    Fact_orders,
    Fact_orders[discount] * Fact_orders[quantity]
)

Discount burden =
DIVIDE([Total Discount], [Total Rev before discount], 0)

-- ─────────────────────────────────────────
-- ORDER & CUSTOMER METRICS
-- ─────────────────────────────────────────

Total Num of Orders =
COUNTROWS(Fact_orders)

Num of customers =
DISTINCTCOUNT(Fact_orders[customer_id])

Repeated Customers =
CALCULATE(
    DISTINCTCOUNT(Fact_orders[customer_id]),
    HAVING(COUNTROWS(Fact_orders), > 1)
)

One order customers =
[Num of customers] - [Repeated Customers]

Repeat Rate % =
DIVIDE([Repeated Customers], [Num of customers], 0)

-- ─────────────────────────────────────────
-- RATING
-- ─────────────────────────────────────────

Avg Rating =
AVERAGE(Fact_orders[rating])
```

---

## 🖥️ Power BI Dashboard Pages

All 5 pages share the same restaurant-themed color palette (dark brown + warm amber) and consistent sidebar filters for **Branch**, **Category**, **Quarter**, and **Year**.

---

### 1. 📋 Executive Overview

> High-level business summary for leadership — revenue, orders, ratings, and branch/category performance.

<img width="1402" height="791" alt="Executive overview" src="https://github.com/user-attachments/assets/98dfcb89-696b-4fd4-ba53-031d619a025d" />


**Highlights:**
- Total Revenue: **$2.898bn** | Avg Order Value: **$261** | Total Orders: **11.110M** | Avg Rating: **3.7**
- **Cairo** dominates with **$1.014bn** in revenue — nearly double the next branch (Giza at $579M)
- Revenue is remarkably **stable across all 12 months** (~$238M–$246M), indicating strong demand consistency
- **Mashweyat (مشويات)** leads in revenue share at **35%** of total
- Revenue per category is almost uniform (~2.2M orders each), suggesting balanced menu performance

---

### 2. 💰 Sales & Revenue Analysis

> Breakdown of revenue streams by payment method, order type, price per item, and year-over-year trends.

<img width="1405" height="790" alt="Sales   Revenue Rnalysis" src="https://github.com/user-attachments/assets/39fb16ce-e1c6-4a48-ad40-14e3bd0510a9" />


**Highlights:**
- Total Net Revenue: **$2.898bn** | Gross Revenue (before discount): **$3.063bn**
- Total Discount Given: **$401.4K** | Avg Price per Order: **$86**
- **Dine-in** is the leading order type at **39.99%** (4M orders)
- **Cash** is the most used payment method with **5.6M** transactions
- Revenue trend from **2020–2025** is nearly flat (~$482M–$484M/year) — very stable but low growth
- **Kofta & Kabab** command the highest avg price at **$150/item**

---

### 3. 🕐 Time Intelligence

> Order volume and rating trends broken down by hour, time slot, day of week, and month.

<img width="1398" height="791" alt="Time Intelligence" src="https://github.com/user-attachments/assets/042b3525-8767-4591-aa36-34848858539b" />


**Highlights:**
- Total Revenue: **$2.898bn** | Avg Order Value: **$261** | Orders: **11.110M**
- **Dinner (18:00–22:00)** is the busiest time slot with **~4M orders**
- **Lunch** follows with **~3.3M orders**; **Late night** and **Breakfast** are least popular
- Order volume is **almost identical across all 7 days** (~1.585M–1.588M), with a slight dip on Tuesday
- **Saturday** is marginally the busiest day
- Avg rating fluctuates throughout the day — peaks appear around hour 14 (2 PM)
- Monthly order volume is flat (~0.91M–0.94M per month) — no strong seasonality

---

### 4. 🍕 Menu Analysis

> Item and category deep-dive — revenue, average price, and discount behavior by menu item.

<img width="1400" height="788" alt="Menu Analysis" src="https://github.com/user-attachments/assets/40b81be0-d298-46f9-be68-01c995621ab3" />


**Highlights:**
- **15 distinct menu items** across 5 categories
- **Kabab (كباب)** is the top-revenue item at **$389K** with avg order value of **$439**
- **Shay (شاي / Tea)** has the lowest revenue at **$16K** with avg price of just **$27**
- **Mashweyat (مشويات)** category has the highest avg price at **$150**; **Mashrobat (مشروبات)** is lowest at **$30**
- Discount is highest on premium items — **Kofta** leads with the most total discount exposure
- Items like **Tahina** and **Baba Ghanoug** show unusually high avg order values relative to their base price, suggesting frequent bulk/combo ordering

---

### 5. 👥 Customer Behavior

> Loyalty, repeat rate, order volume by customer, and behavioral patterns across time and order type.

<img width="1397" height="791" alt="Customer Behavior" src="https://github.com/user-attachments/assets/19b1335a-38ff-4769-8058-952a2a8b6383" />


**Highlights:**
- **200K unique customers** | **100% repeat customer rate** — every customer has ordered more than once
- Top customer (ID: 41263) placed **94 orders** with **315 items** and avg rating of **3.70**
- **Dine-in** leads with **4.442M orders**, Takeaway and Delivery are equal at **~3.334M** each
- Peak order hours are around **14:00 and 20:00**, with a dip at **16:00**
- For the **top 100 customers**, Mashweyat generates the highest order count but Mahashi generates the highest avg order value (**$449**)

---

## 💡 Key Insights

### 🔴 Observations
1. **Revenue is extremely stable** — virtually no seasonality across months or years. This may indicate synthetic data patterns or a very mature, saturated market.
2. **All customers are repeat customers** — 100% repeat rate suggests the customer base is entirely loyal (or a data characteristic of the synthetic dataset).
3. **Cairo dominates** revenue by a wide margin (~$1bn vs ~$580M for next branches) — likely reflecting population density.
4. **Discount burden is minimal** ($401K on $3.06bn gross) — very low discount pressure on margins.

### 🟢 Opportunities
1. **Breakfast and Late Night** time slots are dramatically underperforming — targeted promotions could capture volume.
2. **Tuesday** shows the lowest order volume — an opportunity for mid-week deals.
3. **Mashrobat (beverages)** has the lowest avg price — bundling with main dishes could increase ticket size.
4. **Delivery and Takeaway** are equal in volume but may differ in profitability — worth tracking separately.

---

## 📁 Repository Structure

```
Restaurant-Analytics-Databricks/
│
├── 📂 sql/
│   ├── restaurant_analytics.sql    -> all SQL queries(Data Ingestion, Dimension Tables and Fact Table)
│
├── 📂 screenshots/
│   ├── Schema.png
│   ├── Table.png
│   ├── Measures.png
│   ├── Executive_overview.png
│   ├── Sales___Revenue_Analysis.png
│   ├── Time_Intelligence.png
│   ├── Menu_Analysis.png
│   ├── Customer_Behavior.png
│   ├── Merge_csv_files.png
│   ├── Merge_json_files.png
│   ├── Merge_all_files.png
│   ├── Create_Dim_branch.png
│   ├── Create_Dim_category.png
│   ├── Create_Dim_date.png
│   ├── Create_Dim_item.png
│   ├── Create_Dim_payment_method.png
│   ├── Creatre_Dim_order_type.png
│   ├── Create_Fact_orders_1.png
│   └── Create_Fact_orders_2.png
│
└── README.md
```

---

## 🛠️ Tools & Technologies

| Tool | Purpose |
|---|---|
| **Databricks** | Primary compute — Spark SQL notebooks for ETL & modeling |
| **Apache Spark SQL** | Distributed SQL processing at scale (11M+ rows) |
| **Delta Lake** | ACID-compliant storage format for all tables |
| **Fivetran** | Automated connector: Google Drive → Databricks |
| **Google Drive** | Source file storage (7 CSV + 2 JSON files) |
| **Power BI** | Dashboard and visualization layer |
| **DAX** | Calculated measures in Power BI |

---

## 🚀 How to Reproduce

1. **Upload source files** to Google Drive (7 CSV + 2 JSON)
2. **Connect Google Drive to Fivetran** and sync to your Databricks workspace
3. **Run SQL notebooks** in order (01 → 10) inside Databricks
4. **Connect Power BI** to Databricks via the native connector
5. **Import the `_Measures` table** and recreate DAX measures
6. **Build dashboard pages** using the schema relationships

> ⚠️ Requires a Databricks workspace with Unity Catalog or `workspace.default` schema access.

---

## 👤 Author

**Mohamed Sayed Taha**  
Data Analyst | Databricks | Power BI | SQL | Python | Spark | DAX

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/mohamed-sayed71)
[![Email](https://img.shields.io/badge/Email-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:mohamedsaidtaha1@gmail.com)

---

> ⭐ If you found this project useful, please consider giving it a star!
