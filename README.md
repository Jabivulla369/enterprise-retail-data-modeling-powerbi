# End-to-End Enterprise Retail Data Modeling & Architecture Project

## 📌 Project Overview

In real-world data environments, data rarely arrives perfectly structured for analysis. It usually sits spread across chaotic operational flat files, transactional trackers, and messy schemas.

This project documents the transformation of a fragmented corporate retail dataset into a production-ready, high-performance **Star Schema Data Model** inside Power BI. Following fundamental data warehousing principles, this architecture decouples source data from analytical environments, ensures absolute quantitative accuracy, optimizes storage overhead, and enables lightning-fast DAX calculations.

---

## 🏗️ Architectural Framework & Core Rules

To protect the data model from structural rot, slow response times, and calculation errors, every phase of development strictly adhered to these engineering rules:

1. **The Star Schema Constraint (`Rule_01`):** All structural designs target a clean Star Schema configuration. A centralized transactional event table (`fact_`) sits at the core, surrounded cleanly by isolated entity context descriptors (`dim_`) with explicit one-to-many relationship paths.
2. **Grain Comprehension Prioritization (`Rule_02`):** No structural transformations or merges occur until the exact analytical grain (what a single row truly represents) is fully isolated and verified.
3. **Columnar Pruning Discipline (`Rule_03`):** Every column must earn its place. If a field does not actively drive a strategic metric, filter, slice, or row-level security requirement, it is systematically dropped in Power Query to protect vertical compression efficiency.
4. **Absolute Balance Verification (`Rule_04`):** Centralized financial totals are recorded before a single change is executed. These baselines are re-audited after every transformation step to guarantee zero metric drift.

### 📐 Standardized Conventions Applied

* **Language Integration:** Standardized to English globally.
* **Casing Convention:** Strict `snake_case` naming format across all queries, keys, and attribute parameters.
* **Table Identification:** `fact_` explicitly flags transaction tables; `dim_` flags dimension context maps.
* **Keys Decoupling:** `_id` isolates natural alphanumeric strings sourced directly from external operational software (e.g., `SAP-200001`). `_key` explicitly designates surrogate internal integers generated inside the data warehouse to build clean relationships.
* **Attribute Labeling:** Cryptic operational codings are renamed to business-friendly names.

---

## 🛠️ Phase-by-Phase Technical Implementation

### Phase 1: Prepare & Explore

Before applying data transformations, the original operational workspace was mapped out inside Power Query. To prevent messy developmental query trees, staging folders were immediately deployed to isolate raw source files from analytical queries.
<img width="2048" height="1280" alt="image" src="https://github.com/user-attachments/assets/d82225c6-a73f-4779-ae66-ec46eb5682fc" />


#### 🔍 Source Schema Structural Audit

* **Entity Context Logs:** `CUST_MASTER.csv`, `Address.csv`, `customer_contacts.csv`, `cities.csv`, `regions.csv`, `subcategories.csv`, and `products.csv`.
* **Transactional Event Logs:** `ORDERS_2025.csv`, `ORDERS_2026.csv`, `order_line_items.csv`, `INVOICES.csv`, `invoice_lines.csv`, `payments.csv`, and `shipments.csv`.
* **Strategic Planning Trackers:** `CAMPAIGN_LOG.csv`, `campaign_skus.csv`, and `sales_targets.csv`.

*Design Decision:* To protect operational integrity, zero direct changes were made to these core staging elements. Every operational view was referenced out into isolated development workspaces.

---

### Phase 2: Building Dimensions

Every distinct business entity was extracted using a three-step cycle: **Collect, Reshape/Consolidate, and Clean.**

```
[Operational Staging Tables] ➔ [Power Query Reference & Consolidation] ➔ [Surrogate Integer Key Generation] ➔ [Clean dim_ Dimension Table]

```

#### 🧑‍💼 1. The Customer Dimension (`dim_customer`)

* **Consolidation:** The flat tables `CUST_MASTER.csv`, `Address.csv`, `customer_contacts.csv`, `cities.csv`, and `regions.csv` were unified. Customer contacts were filtered to isolate the primary operational point-of-contact (`IsPrimary = True`).
* **Data Reshaping:** Tables were merged using the natural relational strings `CustomerID` and `AddressID`. Geographic tiers were flattened out to provide seamless filtering (`region_name` ➔ `country_name` ➔ `city_name` ➔ `street_address`).
* **Optimization:** Dropped internal system columns like `hash_key` and duplicated names. Added an auto-incrementing surrogate integer key column named `customer_key`.

#### 📦 2. The Product Dimension (`dim_product`)

* **Consolidation:** Unified `products.csv` and `subcategories.csv` by splitting the pipe-delimited subcategory fields (`CategorySubcategory`) to resolve structural inconsistencies.
* **Data Reshaping:** Mapped category structures directly against product codes, changing cryptic strings into friendly operational fields.
* **Optimization:** Removed unreferenced supplier variables and technical hash columns. Generated `product_key` as the internal surrogate integer identifier.

---

### Phase 3: Building Facts

Fact tables capture the actual events of the business. The goal here was to define the true grain of operational events while avoiding model bloat.

#### 🛒 1. The Core Sales Transaction Fact (`fact_sales`)

* **Grain Isolation:** Extracted line-item entries from `order_line_items.csv` and mapped them against `ORDERS_2025.csv` and `ORDERS_2026.csv` via `OrderID`. A single row in this fact table represents exactly one line-item inside a customer order.
* **Dimensional Connections:** Swapped natural source strings (`CustomerID`, `ProductCode`) for clean surrogate warehouse integers (`customer_key`, `product_key`).
* **Metric Engineering:** Calculated clean transactional column fields:

  ```text
  line_gross_total = quantity * unit_price
  line_discount_amount = line_gross_total * discount_pct
  line_net_total = line_gross_total - line_discount_amount

* **Validation:** Audited aggregate net outputs across both fiscal years against early benchmark captures to confirm zero record omission or double-counting during row appending.

#### 📢 2. The Campaign Activity Log Fact (`fact_campaign`)

* **Structural Intent:** Standardized the marketing records inside `CAMPAIGN_LOG.csv` to capture daily channel spend, serving as the central table for measuring return on investment (ROI).

#### 📉 3. The Sales Target Factless Table (`fact_sales_target`)

* **Structural Intent:** Mapped strategic monthly expectations from `sales_targets.csv` out against regional fields. Because this table tracks performance metrics without line-item transactions, it operates as a factless fact table designed exclusively for target vs. actual comparison analysis.

---

### Phase 4: Polishing & Optimizing the Model

#### 📅 1. Date Dimension Integration (`dim_date`)

To avoid using heavy, auto-generated hidden tables, a dedicated `dim_date` dimension was built using a custom M script. This table spans every transactional date within the dataset, providing clean columns for fiscal years, quarters, months, weeks, and weekdays to support advanced time-intelligence calculations.

#### 📐 2. Measures Group Setup

To keep the field list organized, all DAX metrics were collected inside a dedicated calculations folder (`_measures`). Core measures include:

```dax
// Financial Performance Baselines
total_gross_revenue := SUM(fact_sales[line_gross_total])
total_discount_given := SUM(fact_sales[line_discount_amount])
total_net_revenue := SUM(fact_sales[line_net_total])
total_items_sold := SUM(fact_sales[quantity])
total_orders_placed := DISTINCTCOUNT(fact_sales[order_id])

// Advanced Time-Intelligence Analysis
net_revenue_ly := CALCULATE([total_net_revenue], SAMEPERIODLASTYEAR(dim_date[date]))
net_revenue_yoy_growth := DIVIDE([total_net_revenue] - [net_revenue_ly], [net_revenue_ly], 0)

// Strategic Marketing Performance Metrics
total_marketing_spend := SUM(fact_campaign[spend])
total_ad_impressions := SUM(fact_campaign[impressions])
total_ad_clicks := SUM(fact_campaign[clicks])
click_through_rate := DIVIDE([total_ad_clicks], [total_ad_impressions], 0)
cost_per_click := DIVIDE([total_marketing_spend], [total_ad_clicks], 0)
return_on_ad_spend := DIVIDE([total_net_revenue], [total_marketing_spend], 0)

```

#### 🔒 3. Data Security Implementation (RLS)

To control data access based on user authorization, Row-Level Security (RLS) was applied to `security.csv` by filtering user email contexts against localized geographic regions:

```dax
[region_name] = CALCULATE(
    MAX(security[Region]),
    security[UserEmail] = USERPRINCIPLENAME()
)

```

This rule filters down the `dim_customer` dimension automatically, which dynamically secures all transactional rows inside `fact_sales` based on the user's assigned territory.

---

## 📊 Final Data Model Architecture Diagram

The final model forms a highly optimized Star Schema configuration. Every relationship forms a clean one-to-many (`1 -> *`) directional filtering flow, keeping lookup tables clear of messy data paths.

```
                    +--------------------+
                    |      dim_date      |
                    +--------------------+
                              | (1)
                              |
                              | (*)
+--------------+    +--------------------+    +---------------+
| dim_customer |(1) |                    | (1)|  dim_product  |
+--------------+----+     fact_sales     +----+---------------+
                (*) |                    | (*)
                    +--------------------+
                              | (*)
                              |
                              | (1)
                    +--------------------+
                    |    fact_campaign   |
                    +--------------------+

```

---

## 📈 Analytical Business Capabilities

With this structure complete, the business can seamlessly run advanced corporate retail analytics, including:

* **Territory Sales Trajectories:** Tracking net revenue growth trends year-over-year across distinct municipal markets.
* **Product Management Matrix:** Isolating low-performing categories and tracking discount amounts against inventory velocity.
* **Marketing Efficiency Analysis:** Mapping return on ad spend (ROAS) across different channels to maximize budget impact.
* **Granular Security Protections:** Dynamic row-level security limits data views by country or region automatically, providing secure access for regional managers.
