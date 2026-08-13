# Atlikon FMCG — Databricks Data Engineering Project

A hands-on **Databricks data engineering project** built using FMCG sales data from Atlikon.
The project demonstrates how raw operational data can be transformed into a clean, analytics-ready data model using **Delta Lake, PySpark, SQL, and the Medallion Architecture**.

The main goal of this project was to practice building an end-to-end data pipeline in Databricks, including **data ingestion, data cleansing, dimensional modeling, full loads, incremental loads, Delta Lake features, and dashboarding**.

---

## 📌 Project Overview

This project simulates an FMCG analytics environment where sales data from a business unit needs to be integrated with a parent company's data model.

The pipeline processes multiple data domains:

* Customers
* Products
* Gross Pricing
* Orders / Sales Transactions
* Date Dimension

The processed data is ultimately used to create an **analytics-ready sales view** and a Databricks dashboard for business reporting.

### High-Level Architecture

```text
                 Source CSV Files
                       │
                       ▼
              ┌─────────────────┐
              │   BRONZE LAYER  │
              │   Raw Delta     │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │   SILVER LAYER  │
              │ Clean + Validate│
              │ + Standardize   │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │    GOLD LAYER   │
              │ Business-ready  │
              │ Data Model      │
              └────────┬────────┘
                       │
             ┌─────────┴──────────┐
             ▼                    ▼
       SQL Analytics         Databricks
           View               Dashboard
```

---

# 🏗️ Medallion Architecture

The project follows the **Medallion Architecture** using three schemas within the `fmcg` catalog.

```text
fmcg
├── bronze
├── silver
└── gold
```

## 🥉 Bronze Layer

The Bronze layer stores the source data in its raw form using **Delta tables**.

The ingestion process also captures metadata such as:

* File name
* File size
* Read timestamp

Delta Change Data Feed is enabled on the ingested tables to support tracking of row-level changes.

Example:

```text
fmcg.bronze.customers
fmcg.bronze.products
fmcg.bronze.gross_price
fmcg.bronze.orders
```

The Bronze layer intentionally keeps the source data close to its original state so that the raw data can be audited and reprocessed when required.

---

# 🥈 Silver Layer

The Silver layer is responsible for **data cleaning, validation, standardization, and transformation**.

Major transformations include:

### Customer Data

* Remove duplicate customer records
* Trim unnecessary spaces
* Correct city-related data quality issues
* Standardize customer names
* Handle missing city values
* Convert customer IDs to the required data type
* Standardize attributes to match the parent-company data model

### Product Data

* Remove duplicate products
* Standardize category names
* Apply title case
* Correct spelling inconsistencies
* Standardize product attributes

For example:

```text
energy bars → Energy Bars
protien → Protein
```

### Pricing Data

Pricing data required additional validation because the source contained inconsistent date and price values.

Transformations include:

* Normalize different date formats
* Convert month values into proper dates
* Validate numeric prices
* Handle negative prices
* Handle invalid / non-numeric price values
* Aggregate pricing at the appropriate product/year level
* Select the latest applicable price

### Orders Data

Order data is cleaned and standardized before being moved to the Gold layer.

Transformations include:

* Handle missing order quantities
* Validate customer IDs
* Standardize order dates
* Join orders with product information
* Convert daily transaction data to the required monthly grain

---

# 🥇 Gold Layer

The Gold layer contains business-ready data designed for analytics and reporting.

The project combines the cleaned FMCG data with the parent-company data model.

Key Gold tables include:

```text
fmcg.gold.dim_customers
fmcg.gold.dim_products
fmcg.gold.dim_gross_price
fmcg.gold.dim_date
fmcg.gold.fact_orders
```

A monthly Date Dimension is also generated for analytics.

The date dimension contains attributes such as:

* Date Key
* Year
* Month
* Month Name
* Month Short Name
* Quarter
* Year Quarter

Example:

```text
202401 → January → Q1 → 2024-Q1
```

---

# 🔄 Full Load & Incremental Load

One of the main objectives of this project was to practice both **full-load and incremental data processing**.

## Full Load

The full-load pipeline reads all available files from the landing location and processes them through the Bronze, Silver, and Gold layers.

```text
Landing
   ↓
Bronze
   ↓
Silver
   ↓
Gold
```

After ingestion, processed source files are moved from the `landing` directory to the `processed` directory.

This prevents already-ingested files from being processed repeatedly.

---

## Incremental Load

The incremental pipeline is designed to process only newly arrived files.

The workflow checks the landing location for new CSV files:

```text
New Files
   ↓
Check Landing
   ↓
Bronze Append
   ↓
Silver Transformation
   ↓
Gold Merge
   ↓
Cleanup / Move Files
```

This approach demonstrates an important real-world data engineering pattern where pipelines should process **only newly arrived data instead of rebuilding the entire dataset every time**.

---

# 🔁 Delta Lake

Delta Lake is used as the storage layer throughout the project.

Some of the Delta Lake concepts practiced include:

* Delta tables
* ACID transactions
* Change Data Feed
* MERGE operations
* Append operations
* Overwrite operations
* Schema handling
* Metadata tracking
* Time-travel capable storage

Change Data Feed is enabled during ingestion:

```python
.option("delta.enableChangeDataFeed", "true")
```

This allows changes to data to be tracked at the row level and provides a foundation for more advanced incremental processing.

---

# 🔗 Parent Company Integration

A major part of this project is integrating the FMCG business-unit data with the **Atlikon parent-company data model**.

The child/source data and parent data have differences in:

* Data granularity
* Column names
* Data types
* Attribute naming
* Product/customer structures

The pipeline standardizes the source data before merging it into the parent-company Gold model.

One important example is the sales grain.

The source sales data is available at a **daily level**, while the analytics model requires sales at a **monthly level**.

```text
Daily Sales Data
       ↓
Aggregation
       ↓
Monthly Sales Data
       ↓
Parent Company Gold Model
```

---

# 📊 Analytics View

An enriched SQL view is created for dashboarding:

```sql
fmcg.gold.vw_fact_orders_enriched
```

The view combines:

```text
Fact Orders
     │
     ├── Date Dimension
     ├── Customer Dimension
     ├── Product Dimension
     └── Gross Price Dimension
```

It provides business-friendly fields such as:

### Dimensions

* Date
* Year
* Month
* Quarter
* Customer
* Market
* Platform
* Channel
* Division
* Category
* Product
* Variant

### Measures

* Sold Quantity
* Gross Price
* Total Revenue

Revenue is calculated as:

```text
Total Revenue = Sold Quantity × Gross Price
```

This enriched view acts as the primary source for the Databricks dashboard.

---

# 📈 Atlikon Sales Dashboard

The project includes a Databricks dashboard built on top of:

```text
fmcg.gold.vw_fact_orders_enriched
```

The dashboard provides high-level sales KPIs including:

* **Total Revenue (INR)**
* **Total Units Sold**
* **Total Orders**
* **Average Order Value**

The dashboard can be used to analyze sales performance across dimensions such as:

* Time
* Product
* Category
* Customer
* Market
* Platform
* Channel

---

# 🗂️ Project Structure

```text
sportsbar_fmcg/
│
├── Atlikon Sales Dashboard.lvdash.json
├── dashboard_sample_image.png
├── README.md
│
└── consolidated_pipeline/
    │
    ├── setup/
    │   ├── 1_setup_catalog.ipynb
    │   ├── dim_date_table_creation.ipynb
    │   └── utilities.ipynb
    │
    ├── dimension_data_processing/
    │   ├── 1_customer_data_processing.ipynb
    │   ├── 2_products_data_processing.ipynb
    │   └── 3_pricing_data_processing.ipynb
    │
    ├── fact_data_processing/
    │   ├── 1_full_load_fact.ipynb
    │   └── 2_incremental_load_fact.ipynb
    │
    ├── View for dashboarding.dbquery.ipynb
    └── incremental_data_parent_company.dbquery.ipynb
```

---

# 🛠️ Technologies Used

| Technology                           | Purpose                                   |
| ------------------------------------ | ----------------------------------------- |
| **Databricks**                       | Data engineering and analytics platform   |
| **Apache Spark / PySpark**           | Distributed data processing               |
| **SQL**                              | Data transformation and analytics         |
| **Delta Lake**                       | Reliable lakehouse storage                |
| **AWS S3**                           | Source / landing / processed data storage |
| **Databricks Workflows / Notebooks** | Pipeline orchestration and processing     |
| **Databricks SQL Dashboard**         | Business reporting and visualization      |

---

# 🧠 Key Data Engineering Concepts Practiced

This project was primarily built as a learning exercise to get hands-on experience with modern data engineering concepts.

### Architecture

* [x] Medallion Architecture
* [x] Bronze / Silver / Gold layers
* [x] Dimensional modeling
* [x] Fact and dimension tables

### Data Ingestion

* [x] CSV ingestion
* [x] S3 data ingestion
* [x] File metadata
* [x] Landing and processed folders
* [x] Full-load processing
* [x] Incremental processing

### Data Transformation

* [x] PySpark DataFrame transformations
* [x] Data cleansing
* [x] Duplicate handling
* [x] Null handling
* [x] Data type conversion
* [x] String standardization
* [x] Date parsing
* [x] Data validation
* [x] Joins and aggregations
* [x] Window functions

### Delta Lake

* [x] Delta tables
* [x] Change Data Feed
* [x] Append
* [x] Overwrite
* [x] MERGE
* [x] Time-travel capable architecture

### Analytics

* [x] SQL views
* [x] Business metrics
* [x] Revenue calculations
* [x] Databricks dashboards

---

# 🚀 Pipeline Flow

The overall pipeline can be summarized as:

```text
                 ┌─────────────────┐
                 │    Source CSV   │
                 │      Files      │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Bronze      │
                 │  Raw Delta Data │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Silver      │
                 │ Clean / Validate│
                 │   Standardize   │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │      Gold       │
                 │ Business Model  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │ Enriched SQL    │
                 │      View       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    Dashboard    │
                 │ Sales Analytics │
                 └─────────────────┘
```

---

# 🎯 Learning Outcomes

Through this project, I practiced designing a complete data pipeline rather than focusing only on individual Spark transformations.

The main areas of learning were:

1. Designing a **Medallion Architecture** in Databricks.
2. Building Bronze, Silver, and Gold Delta tables.
3. Performing data-quality transformations using PySpark.
4. Handling inconsistent real-world source data.
5. Building fact and dimension tables.
6. Implementing full and incremental data loads.
7. Using Delta Lake Change Data Feed.
8. Moving files between landing and processed locations.
9. Integrating data with a parent-company data model.
10. Creating analytics-ready SQL views.
11. Building a business dashboard on top of the Gold layer.

---

# ⚠️ Note on the Project

This is a **personal learning / practice project** to gain hands-on experience with Databricks and modern data engineering workflows.

The project uses FMCG sales data in an Atlikon business scenario and focuses primarily on **data engineering concepts, pipeline design, transformation logic, and analytics**.

---

## 👩‍💻 Project Purpose

**Built as a hands-on Databricks data engineering project to practice designing an end-to-end FMCG data pipeline using PySpark, SQL, Delta Lake, Medallion Architecture, incremental processing, and Databricks dashboards.**
