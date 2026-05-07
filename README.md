# 🍢 Restaurant Analytics — End-to-End Data Pipeline & BI Dashboard
 
<div align="center">
![Project Banner](https://img.shields.io/badge/Project-Restaurant%20Analytics-F07820?style=for-the-badge&logo=databricks&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-2E7D32?style=for-the-badge)
![Track](https://img.shields.io/badge/ITI-Data%20Analysis%20Track-1A1A1A?style=for-the-badge)
 
[![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat-square&logo=databricks&logoColor=white)](https://databricks.com)
[![Delta Lake](https://img.shields.io/badge/Delta%20Lake-003366?style=flat-square&logo=delta&logoColor=white)](https://delta.io)
[![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat-square&logo=powerbi&logoColor=black)](https://powerbi.microsoft.com)
[![SQL](https://img.shields.io/badge/SQL-4479A1?style=flat-square&logo=postgresql&logoColor=white)](https://www.sql.org)
 
</div>
---
 
## 📌 Project Overview
 
A full **end-to-end data pipeline and business intelligence project** built for a restaurant chain operating across **6 Egyptian cities**. The project covers every stage of the data lifecycle — from raw file ingestion on **Databricks** to an interactive **Power BI dashboard** published to the cloud.
 
The dashboard is branded with **أبو ربيع** (Abu Rabee), a real restaurant from Damanhour, Egypt, to make the project as close to a real-world business scenario as possible.
 
> 🎓 **Graduation project** — ITI (Information Technology Institute) · Ministry of Communications · Data Analysis Track
 
---
 
## 🏗️ Architecture — Pipeline Stages
 
```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   SOURCE     │───▶│  INGESTION   │───▶│  DELTA LAKE  │
│              │    │              │    │              │
│ 7 CSV files  │    │ Databricks   │    │ Unified raw  │
│ 2 JSON files │    │ DESCRIBE     │    │ table        │
│ Google Drive │    │ CAST + UNION │    │ ACID support │
└──────────────┘    └──────────────┘    └──────┬───────┘
                                               │
┌──────────────┐    ┌──────────────┐    ┌──────▼───────┐
│   POWER BI   │◀───│ STAR SCHEMA  │◀───│   CLEANING   │
│              │    │              │    │              │
│ DirectQuery  │    │ 1 Fact table │    │ SQL-based    │
│ 5 pages      │    │ 6 Dim tables │    │ Null, Dup,   │
│ 20+ DAX      │    │ dim_date ⭐  │    │ Negatives    │
└──────────────┘    └──────────────┘    └──────────────┘
```
 
---
 
## 📂 Dataset Description
 
| Property | Details |
|---|---|
| **Source** | Google Drive (connected to Databricks) |
| **Format** | 7 CSV files + 2 JSON files |
| **Period** | 2020 – 2025 |
| **Branches** | 6 Egyptian cities |
| **Columns** | order_id, order_date, hour, category, item_name, price, quantity, discount, total_amount, branch, payment_method, order_type, customer_id, rating, is_weekend |
 
### Branches
| City | City (AR) |
|---|---|
| Cairo | القاهرة |
| Giza | الجيزة |
| Alexandria | الإسكندرية |
| Mansoura | المنصورة |
| Tanta | طنطا |
| Assiut | أسيوط |
 
### Categories & Items
The menu includes: **مشويات** (Grills) · **مشروبات** (Beverages) · **مقبلات** (Appetizers) · **محاشي** (Stuffed dishes) · **طواجن** (Casseroles)
 
---
 
## 🔧 Stage 1 — Data Ingestion (Databricks)
 
Connected **Google Drive** to Databricks and loaded all 9 files. The main challenge was that identical columns had **different data types** between CSV and JSON sources.
 
**Steps taken:**
1. Used `DESCRIBE` to inspect schema of each file
2. Applied `CAST` on every column to unify data types
3. Merged CSV files into `restaurant_csv` table
4. Merged JSON files into `restaurant_json` table
5. Applied `UNION ALL` to combine both into one raw table
```sql
-- Example: Unify types and UNION ALL
CREATE OR REPLACE TABLE default.restaurant_raw AS
SELECT
  CAST(order_id     AS BIGINT) AS order_id,
  CAST(order_date   AS DATE)   AS order_date,
  CAST(hour         AS INT)    AS hour,
  CAST(price        AS DOUBLE) AS price,
  CAST(total_amount AS DOUBLE) AS total_amount,
  ...
FROM default.restaurant_csv
 
UNION ALL
 
SELECT
  CAST(order_id     AS BIGINT) AS order_id,
  ...
FROM default.restaurant_json;
```
 
---
 
## 🔷 Stage 2 — Delta Lake
 
The unified table was created as a **Delta Lake** table to:
- Support **ACID transactions** — guarantees data consistency when merging CSV and JSON
- Enable **schema evolution** — handles format differences between sources
- Allow **time travel** — ability to roll back to previous versions
- Work natively with **Databricks + Power BI DirectQuery**
---
 
## 🧹 Stage 3 — Data Cleaning (SQL)
 
Chose **SQL over Python** to go deeper into query optimization and set-based operations.
 
### Issues Found & Fixed
 
| Issue | Detection | Fix |
|---|---|---|
| Negative `price` | `SUM(CASE WHEN price < 0 ...)` | `ABS(price)` |
| Negative `total_amount` | Same pattern | `ABS(total_amount)` |
| Wrong `discount` values (>100%) | `SUM(CASE WHEN discount > 1 ...)` | `discount / 100.0` |
| Duplicate `order_id` | `COUNT(*) - COUNT(DISTINCT order_id)` | `ROW_NUMBER() OVER (PARTITION BY order_id)` |
| Out-of-range `rating` | `MIN/MAX check` | `CASE WHEN rating < 1 THEN 1 WHEN rating > 5 THEN 5` |
| NULL values in key columns | `SUM(CASE WHEN col IS NULL ...)` | `WHERE col IS NOT NULL` filter |
 
```sql
-- Deduplication using ROW_NUMBER
WITH deduplicated AS (
  SELECT *,
    ROW_NUMBER() OVER (
      PARTITION BY order_id
      ORDER BY order_date DESC
    ) AS rn
  FROM default.restaurant_raw
)
SELECT ...
FROM deduplicated
WHERE rn = 1
  AND order_id IS NOT NULL
  AND price    IS NOT NULL;
```
 
---
 
## ⭐ Stage 4 — Data Modeling (Star Schema)
 
Separated the flat table into a proper **Star Schema** to:
- Reduce data redundancy
- Speed up Power BI queries (especially with DirectQuery)
- Enable Time Intelligence functions via `dim_date`
- Support cross-table filtering through proper relationships
### Schema Diagram
 
```
                        dim_customer
                             │ 1
                             │
dim_date ─── 1 ─── * ─── fact_orders ─── * ─── 1 ─── dim_item
                             │
              ┌──────────────┼──────────────┐
              │              │              │
           dim_branch   dim_payment    dim_order_type
```
 
### Tables
 
| Table | Type | Key Columns |
|---|---|---|
| `fact_orders` | Fact | order_id (PK), date_id, branch_id, item_id, customer_id, payment_id, order_type_id, price, quantity, discount, total_amount, rating |
| `dim_date` | Dimension | date_id (PK), year, month, month_name, day, day_name, day_of_week, quarter_label, is_weekend |
| `dim_branch` | Dimension | branch_id (PK), branch_name |
| `dim_item` | Dimension | item_id (PK), item_name, category |
| `dim_customer` | Dimension | customer_id (PK), avg_rating, total_orders |
| `dim_payment_method` | Dimension | payment_id (PK), payment_method |
| `dim_order_type` | Dimension | order_type_id (PK), order_type |
 
> **Why `dim_date` matters:** Built with `year`, `month_name`, `day_name`, `day_of_week` to power all Time Intelligence DAX functions (YTD, MoM, YoY, SAMEPERIODLASTYEAR).
 
---
 
## 📊 Stage 5 — Power BI Dashboard
 
### Connection
Used **DirectQuery** mode (not Import) because the dataset is large — this ensures Power BI always queries fresh data directly from Databricks.
 
### Relationships
Built in **Model View** using Primary Key → Foreign Key connections between all 7 tables.
 
### DAX Measures (20+)
 
Measures are organized into **7 folders** inside a dedicated `#KPIs` table:
 
| Folder | Measures |
|---|---|
| **Main KPIs** | Total Revenue, Total Orders, Avg Order Value, Avg Rating, Total Customers, Total Quantity |
| **Time Intelligence** | Revenue LM, Revenue LY, Revenue YTD, MoM Growth %, YoY Growth % |
| **Branches KPIs** | Top Branch by Revenue, Top Branch by Rating, Avg Revenue per Branch |
| **Menu KPIs** | Best Selling Item, Least Selling Item, Top Category, Total Distinct Items |
| **Customers KPIs** | Top Hour, Top Day, Delivery Orders % |
| **Weekend Analysis** | Weekend Revenue, Weekend Revenue % |
| **Discount Analysis** | Discount Impact |
 
### Dashboard Pages
 
#### 1️⃣ Executive Overview
> KPIs · Revenue by Branch (Bar) · Order Type (Donut) · Payment Method (Donut) · Monthly Revenue (Area Line)
 
**Key numbers:** $652M total revenue · 3M total orders · $261 avg order value · 3.7 avg rating
 
#### 2️⃣ Sales Trend
> MoM & YoY Growth KPIs · Multi-year Area Chart · Detailed Table with Conditional Formatting
 
**Key insight:** 93.28% YoY growth — consistent upward trend across all 5 years.
 
#### 3️⃣ Menu & Items
> Best/Worst Seller Cards · Treemap by Category · Bar Chart Top Items · Scatter (Avg Price vs Orders)
 
**Key insight:** مشويات (Grills) = 39% of total revenue. عصير مانجو leads in order volume (300K orders).
 
#### 4️⃣ Branches
> Branch Performance Cards · Summary Table · Dual-Axis Chart (Revenue + Orders) · Bar Chart by Orders
 
**Key insight:** القاهرة leads with $229M revenue and 875K orders. أسيوط has lowest volume — growth opportunity.
 
#### 5️⃣ Customers & Operations
> Top Hour/Day Cards · Heatmap Matrix (Hour × Day) · Avg Rating by Branch · Order Type Donut
 
**Key insight:** Peak hour = 1 PM. Sunday = highest revenue day. Weekend revenue = 50% of weekly total. Dine-in = 39.97% of all orders.
 
---
 
## 📈 Key Business Insights
 
```
💰 Total Revenue    →  $652 Million
📦 Total Orders     →  3 Million
⭐ Avg Rating        →  3.7 / 5
📈 YoY Growth       →  93.28%
📅 Peak Day         →  Sunday
🕐 Peak Hour        →  1 PM
🏆 Top Branch       →  القاهرة ($229M)
🍽️ Top Item         →  عصير مانجو (300K orders)
🔥 Top Category     →  مشويات (39% revenue)
💳 Top Payment      →  Cash (50%)
🚀 Weekend Revenue  →  50% of weekly total
```
 
---
 
## 🛠️ Tech Stack
 
| Tool | Purpose |
|---|---|
| **Databricks** | Cloud workspace for data engineering |
| **Delta Lake** | ACID-compliant storage format |
| **SQL** | Data exploration, cleaning, modeling |
| **Google Drive API** | Source data connection |
| **Power BI Desktop** | Dashboard development |
| **Power BI Service** | Dashboard publishing |
| **DAX** | Measures and KPI calculations |
 
---
 
## 📁 Repository Structure
 
```
restaurant-analytics-pipeline/
│
├── 📁 sql/
│   ├── 01_ingestion.sql          # CAST + UNION ALL
│   ├── 02_delta_table.sql        # Create Delta Lake table
│   ├── 03_data_cleaning.sql      # All cleaning queries
│   ├── 04_star_schema.sql        # Fact + Dimension tables
│   └── 05_dim_date.sql           # Date dimension
│
├── 📁 powerbi/
│   └── Restaurant_Project.pbix   # Power BI report file
│
├── 📁 screenshots/
│   ├── 01_executive_overview.png
│   ├── 02_sales_trend.png
│   ├── 03_menu_items.png
│   ├── 04_branches.png
│   ├── 05_customers_operations.png
│   └── 06_star_schema_model.png
│
└── README.md
```
 
---
 
## 🚀 How to Reproduce
 
1. **Upload data to Google Drive** and connect to Databricks using the Catalog connector
2. **Run SQL scripts** in order (01 → 05) in a Databricks SQL notebook
3. **Open Power BI Desktop** → Get Data → Databricks → paste your cluster URL
4. **Choose DirectQuery** mode when prompted
5. **Build relationships** in Model View using the PK/FK columns
6. **Create measures** from the DAX folder and organize into display folders
7. **Publish** to Power BI Service
---
 
## 📸 Dashboard Screenshots
 
| Page | Preview |
|---|---|
| Executive Overview | ![p1](screenshots/01_executive_overview.png) |
| Sales Trend | ![p2](screenshots/02_sales_trend.png) |
| Menu & Items | ![p3](screenshots/03_menu_items.png) |
| Branches | ![p4](screenshots/04_branches.png) |
| Customers & Operations | ![p5](screenshots/05_customers_operations.png) |
 
---
 
## 👤 Author
 
**Abdelfatah Gaber**
Data Analysis Track · ITI (Information Technology Institute) · Ministry of Communications, Egypt
 
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin)](https://linkedin.com/in/your-profile)
[![GitHub](https://img.shields.io/badge/GitHub-Follow-181717?style=flat-square&logo=github)](https://github.com/abd-elfatah-gaber)
 
---
 
<div align="center">
*Built with ❤️ in Damanhour, Egypt 🇪🇬*
 
</div>
