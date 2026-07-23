# SQL Data Warehouse & Analytics Project

An end-to-end data analytics project that builds a modern data warehouse in **SQL Server** using the **Medallion Architecture (Bronze → Silver → Gold)**, and delivers business insights through **exploratory data analysis (EDA)**, **SQL-based reporting**, and an interactive **Power BI dashboard**.

This project simulates a real-world business scenario: consolidating data from two separate source systems — a **CRM** and an **ERP** — into a single, clean, analysis-ready data warehouse.

---

## 📌 Project Overview

| | |
|---|---|
| **Database** | SQL Server |
| **Architecture** | Medallion (Bronze / Silver / Gold) |
| **Source Systems** | CRM (customer, product, sales data), ERP (location, customer demographics, product category) |
| **Reporting Layer** | SQL Views (Star Schema) + Power BI Dashboard |
| **Data Format** | CSV files |

---

## 🏗️ Architecture

```
Source Systems (CRM + ERP CSV files)
        │
        ▼
┌─────────────────┐
│   BRONZE LAYER   │  Raw data, loaded as-is via BULK INSERT
└─────────────────┘
        │
        ▼
┌─────────────────┐
│   SILVER LAYER   │  Cleaned, standardized, deduplicated data
└─────────────────┘
        │
        ▼
┌─────────────────┐
│    GOLD LAYER    │  Business-ready Star Schema (Fact & Dimension views)
└─────────────────┘
        │
        ▼
   Power BI Dashboard / SQL Analytics & Reporting
```

**Why Medallion Architecture?**
- **Bronze** — preserves raw source data exactly as received, for traceability and reprocessing
- **Silver** — applies data cleaning, standardization, and business rules
- **Gold** — models data into a Star Schema optimized for reporting and analytics

---

## 🗂️ Data Sources

**CRM System**
- `cust_info.csv` — customer master data
- `prd_info.csv` — product master data
- `sales_details.csv` — sales transactions

**ERP System**
- `LOC_A101.csv` — customer location/country data
- `CUST_AZ12.csv` — customer birthdate & gender
- `PX_CAT_G1V2.csv` — product category, subcategory, and maintenance info

---

## ⚙️ ETL Process

### Bronze Layer (`bronze.load_bronze`)
- Truncates and reloads all bronze tables using `BULK INSERT`
- Loads raw CRM and ERP CSV files with no transformation
- Includes load-duration logging and `TRY/CATCH` error handling

### Silver Layer (`silver.load_silver`)
Key transformations applied:
- **Deduplication** — keeps the most recent customer record using `ROW_NUMBER()`
- **Standardization** — normalizes coded values (e.g., `M`/`F` → `Male`/`Female`, `S`/`M` → `Single`/`Married`)
- **Data validation** — nulls out invalid/future birthdates, fixes malformed date integers
- **Business rule correction** — recalculates sales amount when it doesn't match `quantity × price`; derives missing prices
- **Key parsing** — extracts category ID and clean product key from composite product keys
- **Historization** — calculates product `end_date` using `LEAD()` based on the next start date

### Gold Layer (SQL Views — Star Schema)
- `gold.dim_customers` — customer dimension (merged CRM + ERP data, with fallback logic for gender)
- `gold.dim_products` — product dimension (current/active products only, joined with category data)
- `gold.fact_sales` — sales fact table (linked to dimensions via surrogate keys)

---

## 📊 Analysis Performed

### Exploratory Data Analysis (EDA)
- Database/schema/table exploration
- Measures of scale: total sales, total orders, total quantity, average price
- Time-based exploration: order date range, customer age range
- Magnitude analysis: totals by country, category, gender

### Advanced Analytics
- **Change over time analysis** — monthly/yearly sales trends, running totals
- **Cumulative analysis** — running total of sales over time
- **Performance analysis** — year-over-year product sales vs. average and prior year
- **Part-to-whole analysis** — percentage contribution of each category to total sales
- **Data segmentation** — products segmented into cost ranges; customers segmented into **VIP / Regular / New** based on spending and lifespan
- **Customer Report** (SQL View: `gold.report_customers`) — consolidated customer profile with recency, average order value, and average monthly spend

---

## 📈 Power BI Dashboard

A 3-page interactive Power BI dashboard was built on top of the Gold layer views:

1. **Executive Sales Overview** — KPIs (Total Sales, Orders, Quantity, Avg Order Value), sales trend (with YoY comparison), top products, sales by category and country
2. **Product Analysis** — product-level performance table, top 5 / bottom 5 products by revenue, sales by product line
3. **Customer Analytics** — customer segmentation (VIP/Regular/New), age group distribution, sales by country and gender

**Key DAX features used:**
- Time-intelligence measures (YoY sales growth using `SAMEPERIODLASTYEAR`)
- Calculated columns for customer age, age group, and dynamic customer segmentation (using `RELATEDTABLE`)
- Defensive measures (`+0`, `DIVIDE` with fallback) to handle blank/zero results across filter combinations

---

## 🔍 Key Insight / Data Quality Finding

During dashboard development, the **Components** product category appeared to have zero sales. Rather than assuming this was a bug, the discrepancy was traced back through SQL:

```sql
SELECT p.cat_id, SUM(f.sales_amount) AS total_sales, COUNT(*) AS row_count
FROM silver.crm_sales_details f
JOIN silver.crm_prd_info p ON f.sls_prd_key = p.prd_key
GROUP BY p.cat_id;
```

This confirmed that **Components (e.g., frames, pedals) are never sold as standalone SKUs** — they exist only as sub-parts used to assemble finished bikes. This was a genuine business pattern, not a data pipeline defect, and the dashboard was adjusted (filtered/labeled) to reflect this accurately rather than showing misleading data.

---

## 🛠️ Tech Stack

- **Database:** SQL Server, T-SQL
- **ETL:** Stored Procedures, BULK INSERT, TRY/CATCH error handling
- **Data Modeling:** Medallion Architecture, Star Schema (Fact & Dimension views)
- **SQL Techniques:** CTEs, Window Functions (`ROW_NUMBER`, `LEAD`, `LAG`), Aggregations, Views
- **Visualization:** Power BI, DAX, Power Query

---

## 📁 Project Structure

```
Data-warehouse-sql-project/
│
├── datasets/                          -- Raw source CSV files
│   ├── source_crm/
│   │   ├── cust_info.csv
│   │   ├── prd_info.csv
│   │   └── sales_details.csv
│   └── source_erp/
│       ├── CUST_AZ12.csv
│       ├── LOC_A101.csv
│       └── PX_CAT_G1V2.csv
│
├── docs/                              -- Project documentation
│   ├── data_catalog.md                -- Descriptions of Gold layer tables/columns
│   └── naming_conventions.md          -- Naming standards used across the project
│
├── scripts/                           -- All SQL scripts (DDL + ETL)
│   ├── init_database.sql              -- Creates database and schemas
│   ├── bronze/
│   │   ├── ddl_bronze.sql             -- Bronze layer table definitions
│   │   └── proc_load_bronze.sql       -- Bronze layer ETL stored procedure
│   ├── silver/
│   │   ├── ddl_silver.sql             -- Silver layer table definitions
│   │   └── proc_load_silver.sql       -- Silver layer ETL stored procedure
│   └── gold/
│       └── ddl_gold.sql               -- Gold layer views (Star Schema)
│
├── tests/                             -- Data quality checks
│   ├── quality_checks_silver.sql      -- Validation checks after Silver load
│   ├── quality_checks_gold.sql        -- Validation checks on Gold views
│   └── EDA/                           -- Exploratory data analysis & advanced analytics scripts
│
├── dashboard/                         -- Power BI dashboard file
│   └── powerbi_dashboard.pbix
│
└── README.md
```

---

## 🚀 How to Run

1. Run `scripts/init_database.sql` to create the `DataWarehouse` database and schemas
2. Run the DDL scripts in order to create the Bronze, Silver, and Gold layer tables/views:
   - `scripts/bronze/ddl_bronze.sql`
   - `scripts/silver/ddl_silver.sql`
   - `scripts/gold/ddl_gold.sql`
3. Update the file paths inside `scripts/bronze/proc_load_bronze.sql` to point to your local copies of the CSV files in `datasets/source_crm/` and `datasets/source_erp/`
4. Execute the ETL procedures in order:
   ```sql
   EXEC bronze.load_bronze;
   EXEC silver.load_silver;
   ```
5. (Optional) Run the scripts in `tests/` to validate data quality after each load
6. Query the Gold layer views for analysis (see `tests/EDA/`), or connect Power BI Desktop to the database and select the Gold views to build/refresh the dashboard

---

## 📌 Notes

- This project was built for learning and portfolio purposes, using a sample CRM/ERP dataset
- Data quality issues found during development (e.g., NULL categories, unmatched product keys) were investigated and documented rather than silently ignored, reflecting real-world data analyst workflows
