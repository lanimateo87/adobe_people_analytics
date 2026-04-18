# 07 — Slicer and Filtering Guide

## Model Evolution

This model evolved from a disconnected slicer pattern (IN VALUES DAX) to a relationship-based star schema. Understanding why helps future developers make informed extension decisions.

### Original Design (21 tables)
- 8 separate slicer tables with zero relationships
- Every measure wrapped in `CALCULATE([measure], FILTER(fact_table, fact_table[col] IN VALUES(slicer_table[col]) || NOT ISFILTERED(slicer_table)))`
- Complex DAX, hard to maintain, but guaranteed consistent filtering across all fact tables

### Current Design (16 tables)
- Star schema with dim_employee_v as central hub
- Slicers filter through relationships automatically
- Simple DAX, easy to maintain
- 2 remaining slicer tables exist only where sort columns or unique paths are needed
- USERELATIONSHIP pattern for fact_attrition_v (inactive employee_sk relationship)

## Current Slicer Architecture

### Relationship-Based Slicers (no separate table)

| Slicer Attribute | Source | Filter Path |
|-----------------|--------|-------------|
| Gender | dim_employee_v[gender] | dim_employee_v → fact tables via employee_sk |
| Region | dim_employee_v[region] | dim_employee_v → fact tables via employee_sk |
| Employment Type | dim_employee_v[employment_type] | dim_employee_v → fact tables via employee_sk |
| Job Level | dim_job_v[job_level] | dim_job_v → dim_employee_v → fact tables |
| Job Family | dim_job_v[job_family] | dim_job_v → dim_employee_v → fact tables |
| Comp Band | dim_comp_band_v[band_name] | dim_comp_band_v → dim_employee_v → fact tables |
| Business Unit | dim_department_v[business_unit] | dim_department_v → dim_employee_v → fact tables, AND dim_department_v → fact_budget_v |
| Department | dim_department_v[dept_name] | Same as Business Unit |
| Fiscal Year | dim_date_v[fiscal_year_label] | dim_date_v → fact tables via date_sk |
| Fiscal Quarter | dim_date_v[fiscal_quarter_label] | dim_date_v → fact tables via date_sk |

### Dedicated Slicer Tables (2 remaining)

| Table | Rows | Why It Exists | Connected To |
|-------|------|---------------|-------------|
| slicer_tenure_v | 5 | tenure_sort column needed — "< 1 Year" sorts wrong alphabetically | dim_employee_v[tenure_band_today] |
| slicer_exit_reason_v | 7 | Attribute lives on fact_attrition_v, not dim_employee_v — needs its own relationship path | fact_attrition_v[termination_reason] |

## Slicer Behavior by Fact Table

| Slicer | fact_headcount_v | fact_compensation_v | fact_performance_v | fact_attrition_v | fact_budget_v |
|--------|-----------------|--------------------|--------------------|------------------|--------------|
| Gender | ✓ via relationship | ✓ via relationship | ✓ via relationship | ✓ via USERELATIONSHIP | ✗ no path |
| Region | ✓ | ✓ | ✓ | ✓ via USERELATIONSHIP | ✗ |
| Job Level | ✓ | ✓ | ✓ | ✓ via USERELATIONSHIP | ✗ |
| Business Unit | ✓ | ✓ | ✓ | ✓ via USERELATIONSHIP | ✓ via dim_department_v |
| Department | ✓ | ✓ | ✓ | ✓ via USERELATIONSHIP | ✓ via dim_department_v |
| Fiscal Year | ✓ | ✓ | ✓ | ✓ via date_sk | ✓ via date_sk |
| Exit Reason | ✗ | ✗ | ✗ | ✓ direct | ✗ |

**Key insight:** fact_budget_v only responds to Business Unit, Department, and date slicers. Employee attribute slicers (gender, level, region) do not filter budget data. This is correct — budgets are planned at the department level.

## The USERELATIONSHIP Pattern

fact_attrition_v has two relationships:
- **Active:** fact_attrition_v[date_sk] → dim_date_v[date_sk] (date filtering works automatically)
- **Inactive:** fact_attrition_v[employee_sk] → dim_employee_v[employee_sk] (must be activated per measure)

Why inactive? An active relationship to both dim_date_v and dim_employee_v created an ambiguous path through dim_department_v. Making the employee relationship inactive eliminates the ambiguity while USERELATIONSHIP activates it only during measure evaluation.

Every attrition measure includes:
```dax
CALCULATE(
    COUNTROWS(fact_attrition_v),
    USERELATIONSHIP(fact_attrition_v[employee_sk], dim_employee_v[employee_sk]),
    ...additional filters...
)
```

## Cumulative Measures for Trend Charts

When a line chart has month on the X axis, each data point evaluates in its own month's filter context. Standard measures show per-month values (8 exits, 16 exits, 13 exits). Cumulative measures use the REMOVEFILTERS pattern to build running totals (8, 24, 37):

```dax
VAR CurrentMonth = MAX(dim_date_v[yyyymm])
RETURN
    CALCULATE(
        [base measure],
        REMOVEFILTERS(dim_date_v),
        dim_date_v[yyyymm] <= CurrentMonth,
        VALUES(dim_date_v[fiscal_year])
    )
```

- `REMOVEFILTERS(dim_date_v)` clears the chart axis filter
- `dim_date_v[yyyymm] <= CurrentMonth` rebuilds the cumulative window
- `VALUES(dim_date_v[fiscal_year])` preserves the fiscal year slicer selection
- Does NOT include `VALUES(dim_date_v[fiscal_quarter])` — this would cause quarterly resets

## How to Add a New Slicer

**If the attribute exists on dim_employee_v or a connected dimension:**
1. Add a slicer visual using the column directly (e.g., dim_employee_v[ethnicity])
2. No new table, relationship, or DAX changes needed
3. Test: verify all measures on the page respond to the slicer

**If the attribute needs a custom sort order:**
1. Create a slicer view in `04_build_semantic_layer`:
```sql
CREATE OR REPLACE VIEW semantic.slicer_newattr_v AS
SELECT DISTINCT attribute, sort_order FROM source_table ORDER BY sort_order
```
2. Load into Power BI
3. Create relationship: dim_employee_v[attribute] → slicer_newattr_v[attribute], *:1, Single
4. Set Sort By Column in Power BI
5. Add slicer visual using slicer_newattr_v[attribute]

**If the attribute lives on a fact table with no dimension path:**
1. Create a slicer view (like slicer_exit_reason_v)
2. Connect directly to the fact table
3. This is rare — most attributes belong on dimensions

## Slicer Usage by Report Page

| Page | Slicers Used |
|------|-------------|
| Executive Summary | Fiscal Year, Fiscal Quarter, Business Unit, Department, Region |
| Headcount & Org Health | Same + Employment Type |
| Attrition & Retention | Same + Exit Reason, Gender |
| Compensation & Pay Equity | Same + Gender, Job Level |
| Performance & Talent | Same + Review Cycle, Gender |
| Workforce Planning | Fiscal Year, Fiscal Quarter, Business Unit, Department |
| Diversity & Inclusion | Same + Job Level |
