# Adobe People Analytics — Semantic Layer Portfolio

## The Problem

20+ Power BI dashboards break whenever Data Engineering changes source tables during the SAP HANA → Databricks migration. The senior BI developer loses days fixing reports after each schema change. Metric definitions conflict across business partners. Documentation is sparse or missing.

## The Solution

A semantic contract layer in Databricks — 16 views in the `semantic` schema that Power BI connects to exclusively. When Data Engineering renames a column, I update one view alias. Zero dashboards break.

## Architecture

```
SAP HANA / Workday
       ↓
┌─────────────────────────────────────┐
│  DATA ENGINEERING OWNS              │
│  Bronze (raw) → Silver (cleaned)    │
├─────────────────────────────────────┤
│  PEOPLE ANALYTICS OWNS             │
│  Gold (facts) → Semantic (contracts)│
│       ↓                             │
│  Power BI (16 tables, 78 measures)  │
└─────────────────────────────────────┘
```

The ownership boundary is explicit: Data Engineering delivers clean Silver tables. People Analytics transforms them into stable contracts that Power BI consumes.

## What Each Layer Demonstrates

| Layer | Purpose | Key Decision |
|-------|---------|--------------|
| Bronze | Raw source data, 5,000 employees with SCD2 history | Dynamic date range aligned to Adobe fiscal year (Dec start) |
| Silver | Cleaned dimensions, surrogate keys, SCD2 integrity | employee_sk (not emp_id) prevents many-to-many in Power BI |
| Gold | Fact tables, pre-aggregated trends | Time comparisons computed in SQL, not DAX |
| Semantic | 16 contract views Power BI connects to | Column aliases absorb upstream renames |
| Power BI | 16-table star schema, 78 DAX measures | USERELATIONSHIP pattern for attrition, cumulative measures |

## Technical Decisions

- **SCD2 surrogate key**: `employee_sk` created via `ROW_NUMBER()` in Silver. Each SCD2 version gets a unique integer, converting many-to-many into clean one-to-many relationships.
- **Star schema over disconnected slicers**: Evolved from 21 tables with 8 disconnected slicer tables (IN VALUES pattern) to 16 tables with relationship-based filtering. Simpler DAX, same analytical coverage.
- **USERELATIONSHIP for attrition**: `fact_attrition_v` uses an inactive relationship to `dim_employee_v` activated per-measure. Avoids ambiguous paths while allowing employee attribute slicers to filter attrition data.
- **Cumulative measures**: Running totals use `REMOVEFILTERS(dim_date_v)` with `VALUES(dim_date_v[fiscal_year])` to accumulate within fiscal year without resetting at quarter boundaries.
- **Adobe fiscal year alignment**: December = FY start. All date dimensions, budget data, and measures align to Adobe's fiscal calendar.
- **Dynamic date filtering**: `dim_date_v` automatically filters to last 5 fiscal years through today. Bronze data generation adapts to current date.

## Report Pages

1. **Executive Summary** — KPI cards with YoY comparisons, headcount trend, cumulative attrition rate, actual vs approved headcount by business unit
2. **Headcount & Org Health** — Waterfall, matrix, treemap, span of control
3. **Attrition & Retention** — Cumulative exit trends, exit reasons, tenure at exit
4. **Compensation & Pay Equity** — Gender pay gap by job level (~6% engineered gap at IC4+), compa ratio analysis
5. **Performance & Talent** — Rating distribution, high performer rate, flight risk quadrant
6. **Workforce Planning** — Approved vs actual headcount and payroll by department
7. **Diversity & Inclusion** — Gender representation by level, pay gap trends, promotion rate equity

## Validation

11 automated data quality checks — all PASS. Results logged to `silver.reconciliation_log` with UUID run IDs and timestamps.

## Project Structure

```
adobe-people-analytics/
├── README.md
├── notebooks/
│   ├── 00_create_catalog_schema.sql
│   ├── 01_generate_sample_data.py
│   ├── 02_build_silver_layer.sql
│   ├── 03_build_gold_layer.sql
│   ├── 04_build_semantic_layer.sql
│   └── 05_validation.sql
├── docs/
│   ├── 01_architecture_overview.md
│   ├── 02_view_inventory.md
│   ├── 03_data_dictionary.md
│   ├── 04_change_management_runbook.md
│   ├── 05_dax_measures_library.md
│   ├── 06_validation_log.md
│   └── 07_slicer_pattern_guide.md
└── pbix/
    └── adobe-people-analytics.pbix
```

## Links

- **GitHub**: [https://github.com/lanimateo87/adobe_people_analytics]
- **Power BI Service**: [https://app.powerbi.com/view?r=eyJrIjoiNTIxMTMyMDItZTg1MC00YmZhLWE3ZmEtM2VlNmI1ZGY3Yjc5IiwidCI6IjdkOTI5ZTYxLTlhOTItNGUyNC1iMjQwLWYzNGI3ZjUzYTZlZSIsImMiOjZ9]

## Contact

Lani Mateo — lanimateo@gmail.com
