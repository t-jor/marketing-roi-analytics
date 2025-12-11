# Marketing ROI Modeling & Performance Analysis (PowerFlow Case Study)

**Modeling user-level ROI and evaluating channel efficiency through unified marketing and in-app behavioral data**  
*A modular dbt Cloud pipeline in Snowflake integrating attribution sources, transaction data, and LTV logic into a production-ready ROI data model.*

---

## 📘 Project Overview

This portfolio project simulates a BI case study for the fictional fitness app PowerFlow, focusing on user-level ROI modeling and channel performance analysis using dbt Cloud and Snowflake.

### 💼 Business Context

**PowerFlow** is a digital fitness app offering subscription-based workout plans with guided video instructions.  
The **Marketing Department** aims to identify which channels and campaigns deliver the strongest financial returns across the user lifecycle.  
The core business goal is to understand **which marketing channel and/or campaign ID achieves the highest Return on Investment (ROI)** at different points in a user's lifetime — enabling smarter budget allocation and improved user acquisition efficiency.

### 🧠 Analytical Approach

This project translates that business challenge into a reproducible data model built with **dbt Cloud** and **Snowflake**.  
It integrates and transforms multiple data sources into a unified, analytics-ready ROI dataset.

Using the shared `device_id` as the linking key, the models calculate:

- Cost per user and campaign  
- Revenue and profit by channel and cohort  
- ROI trends over time

The dbt pipeline follows a **three-layer architecture** (*staging → intermediate → marts*) with modular SQL logic, automated tests, and clear documentation to ensure data quality and transparency.  
All transformations are version-controlled via GitHub and orchestrated in dbt Cloud.

---

## 🧱 Architecture & Tech Stack

| Component | Purpose |
|------------|----------|
| **dbt Cloud** | Transformation and orchestration with environment-based schema logic |
| **Snowflake** | Cloud data warehouse used as target database |
| **GitHub** | Version control for all dbt models and documentation |
| **dbt Packages** | `dbt_utils`, `codegen` for testing and documentation support |
| **dbt Seeds** | `campaign_cost.csv` used to bring static marketing spend data into the pipeline |

---

## 🔄 Data Flow & Model Layers

The data pipeline follows the **standard dbt layered architecture**.
Each layer builds on the previous one, ensuring full traceability from raw data to ROI metrics

```text
sources  →  staging  →  intermediate  →  marts
```

---

## 🧩 Data Sources Overview

The PowerFlow project integrates both **in-app user data** and **marketing attribution data** to enable user-level ROI analysis.  
Both marketing sources (AppsFlyer and Google Ads) track users based on a unique `device_id`.  
In this dataset, each device corresponds to exactly one user — allowing consistent attribution and cost allocation across all tables.

- **AppsFlyer:** provides *user-level acquisition costs* directly within its data, while
- **Google Ads:** only contains *device-level attribution details* (campaign and attribution date). Standardized campaign cost information is therefore added later via a separate seed file.
- **In-app transactions:** delivers user purchase and revenue data.

---

### 🗂️ 1️⃣ Sources

The project uses four primary raw input tables — two related to user activity within the app and two related to marketing attribution.  
They are defined in `_sources.yml` and stored in the Snowflake schema `powerflow.public`.

| Source | Description |
|---------|--------------|
| `registrations_raw` | Contains key information on user registrations, including `device_id`, registration date, and country. |
| `transactions` | Records all user purchases and revenue events made within the app. |
| `appsflyer_raw` | Aggregates attribution data for each `device_id` from multiple marketing channels (e.g., TikTok, YouTube, IronSource, AdMob). Includes *user-level cost data* for each acquired user. |
| `google_ads` | Provides *device-level attribution data* (campaign and attribution date) for users acquired via Google Ads. Does **not** include cost information — costs are joined later from the seed file. |

> ⚙️ All raw tables serve as the foundation for cleaning and transformation in the staging layer.  
> 🕒 **Note:** The `attribution_date` in marketing tables represents the date of the ad interaction that led to the app install.  
> It may differ from the `registration_date` in user data, which captures when the user actually registered in the app.

---

### 💾 Seed File

An additional **seed file** supplements the missing cost information for Google Ads campaigns:

| Seed | Description |
|-------|--------------|
| `campaign_cost.csv` | Contains standardized *cost per user* for each Google Ads campaign, used to calculate acquisition cost and ROI metrics. |

---

### 2️⃣ Staging Models (`stg_`)

Each source has a dedicated **staging model** to clean, rename, and standardize columns.  
This layer ensures consistent naming conventions and data formats across all sources.

| Model | Description |
|--------|--------------|
| `stg_appsflyer.sql` | Standardizes attribution data from AppsFlyer platform. |
| `stg_google_ads.sql` | Normalizes and enriches Google Ads data – adds channel mapping and standardized campaign costs. |
| `stg_registrations_clean.sql` | Cleans and validates user registration data – keeps only valid registrations. |
| `stg_transactions.sql` | Standardizes transaction data for consistent aggregation. |

All staging models are **materialized as views** for lightweight transformations.

---

### 3️⃣ Intermediate Models (`int_`)

These models combine and enrich data across multiple staging sources.  
They implement business logic such as user attribution, LTV calculations, and campaign mapping.

| Model | Description |
|--------|--------------|
| `int_marketing_attribution.sql` | Links each user to their originating marketing source |
| `int_users_with_attribution.sql` | Consolidates user registration with marketing data - adds organically acquired customers |
| `int_user_ltv.sql` | Calculates cumulative lifetime value (LTV) per user based on transactions |

---

### 4️⃣ Mart Models (`marts/`)

The **final data layer** used for business reporting and analytics.  
Models here are **materialized as tables** for performance and stability.

| Model | Description |
|--------|--------------|
| `user_roi.sql` | Combines user LTV with marketing spend to compute ROI per user |

---

## 💰 ROI Logic

The core output of this project is the **User ROI Table**.

**ROI Calculation:**

```sql
ROI = User_Lifetime_Revenue / Acquisition_Cost
```

For **organically acquired users**, acquisition cost is set to `0`.  
The model handles this via `DIV0()` logic in Snowflake to avoid division errors and maintain meaningful output.

The final table enables comparison of **paid vs. organic** acquisition performance and highlights which marketing channels deliver the highest lifetime return.

---

## 🧪 Testing & Data Quality

Data quality is ensured through a combination of **generic and custom dbt tests**:

| Type | Example |
|------|----------|
| **Generic Tests** | `unique`, `not_null` on keys and categorical columns |
| **Custom Test** | `positive_lifetime.sql` verifies that each user’s lifetime value is positive |
| **YAML Documentation** | `_schema_stg.yml`, `_schema_int.yml`, `_schema_roi.yml` describe columns and purposes |

All tests can be executed via:

```bash
dbt test
```

---

## 🧾 Seeds

The project includes one static seed:

- **`campaign_cost.csv`** – used to join campaign metadata and cost information with attribution results for google ads.

Seeds are loaded via:

```bash
dbt seed
```

---

## ⚙️ Deployment & Orchestration

- The project is deployed in **dbt Cloud** with a connection to **Snowflake**.  
- Branching and version control are managed via **GitHub** (`main` and feature branches).  
- Two environments are defined in dbt Cloud to separate development and production:

  | Environment | DBT_ENV_NAME | Behavior |
  |--------------|--------------|-----------|
  | `Develop_Powerflow` | `dev` | All models are built in the developer’s default schema (`dbt_tjortzig`). |
  | `Prod_Powerflow` | `prod` | Models are deployed into environment-specific schemas as listed below. |

---

### 🏗️ Environment-Specific Schemas

A custom macro (`macros/generate_schema_name.sql`) dynamically assigns schemas based on the active environment.  
This ensures that all development models remain isolated, while production models follow a structured naming convention.

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {% set default_schema = target.schema -%}
    {% set env = env_var('DBT_ENV_NAME') -%}

    {% if custom_schema_name is none or env == 'dev' -%}
        {{ default_schema }}
    {% else -%}
        {{ custom_schema_name | trim }}
    {% endif -%}
{%- endmacro %}
```

**Schema naming in production:**

- `staging` – for staging views  
- `intermediate` – for intermediate transformations  
- `lookup` – for seed- or mapping-based reference tables  
- `marts` – for final analytical tables  

---

### 🧭 Job Execution

**Jobs in dbt Cloud** are *not scheduled* (static training data) to ensure deterministic rebuilds for each full pipeline run.

Example command used for full builds and tests:

```bash
dbt build --select +user_roi
```

---

## 📈 PowerFlow DAG Overview

The following Directed Acyclic Graph (DAG) shows the full lineage of models within the PowerFlow project —
from raw sources and seeds to final analytical tables.  
It highlights the clear separation between staging, intermediate, and mart layers, following dbt’s modular design principles.

<!-- markdownlint-disable MD033 -->
<p align="center">
  <img src="./docs/dag_powerflow.png" alt="PowerFlow dbt DAG" width="85%" style="border: 1px solid #ccc; border-radius: 6px;"/>
</p>
<!-- markdownlint-enable MD033 -->

*Visualization generated from dbt Cloud — each node represents a model or data source in Snowflake.*

---

## 💡 Key Learnings & Best Practices

Throughout this project, several **dbt best practices** were applied:

- ✅ Consistent naming conventions and clear folder structure  
- ✅ One staging model per source for transparency and reusability  
- ✅ Readable SQL (use of CTEs, aliases, and standardized formatting)  
- ✅ Documentation embedded in YAML for every model  
- ✅ Custom test for positive lifetime value validation  
- ✅ Separation of logic into layered models for maintainability
- ✅ Use of environment variables for dynamic schema routing (via custom macro)

These improvements make the project easier to understand, extend, and present as a professional data transformation workflow.

---

## 🚀 Next Steps

These ideas outline how the PowerFlow project could evolve into a production-grade marketing analytics pipeline:

- Adding **channel-level ROI dashboards** in Tableau or Looker Studio  
- Implementing **incremental models** for faster runs on large datasets  
- Expanding attribution logic with **multi-touch models**  
- Integrating real campaign APIs (e.g., Google Ads, Meta) as sources  

---

## 👨‍💻 Author

**Thomas Jortzig**  
Marketing ROI Modeling & Performance Analysis – PowerFlow Case Study (10/2025)
