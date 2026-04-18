# 05 — DAX Measures Library

## Overview

78 measures across 8 display folders, hosted on the `_Measures` calculated table (expression: `{1}`). All measures created via Tabular Editor 2.x Advanced Scripting (C#).

## Design Principles

- **DIVIDE() everywhere** — never use raw `/` operator. Third argument is 0 (returns zero on division by zero)
- **DATEADD requires dim_date_v marked as date table** on the `[date]` column. Without this, time intelligence functions fail
- **No IN VALUES pattern** — the 16-table star schema uses relationship-based filtering. Slicers filter through dim_employee_v relationships automatically
- **USERELATIONSHIP for attrition** — fact_attrition_v has an inactive relationship to dim_employee_v. Every attrition measure activates it explicitly so employee attribute slicers filter attrition data
- **REMOVEFILTERS for cumulative measures** — trend line charts override the visual axis filter using REMOVEFILTERS(dim_date_v) then rebuild the window with VALUES(dim_date_v[fiscal_year]) and yyyymm <= CurrentMonth
- **Percentage measures return decimals** (0.06, not 6). FormatString "0.00%" handles display. Never multiply by 100 in DAX
- **Underscore prefix** (_Active HC Filter, _Latest Month) signals internal helper measures not for direct use in visuals

## 00 Base (4 measures)

| Measure | DAX | Notes |
|---------|-----|-------|
| _Active HC Filter | `CALCULATE(SUM(fact_headcount_v[is_active]), fact_headcount_v[is_active] = 1)` | Internal helper |
| _Latest Month | `CALCULATE(MAX(fact_headcount_v[snapshot_yyyymm]), fact_headcount_v[is_active] = 1)` | Latest month with data |
| _Last Complete Month | `VAR MaxMonth = CALCULATE(MAX(dim_date_v[yyyymm]), fact_headcount_v[is_active] = 1) ...` | Returns last complete month within filter context. If max month = current month, steps back one |
| Report Period Label | `FORMAT(DATE(Y, M, 1), "MMM YYYY")` | Displays "Mar 2026" for KPI card context |

## 01 Headcount (14 + 3 cumulative)

| Measure | Type | Format |
|---------|------|--------|
| Active Headcount | Additive | #,0 |
| Active Headcount LM | Time intelligence | #,0 |
| Active Headcount LY | Time intelligence | #,0 |
| HC MoM Delta | Derived | #,0 |
| HC MoM Change | Ratio | 0.00% |
| HC YoY Delta | Derived | #,0 |
| HC YoY Change | Ratio | 0.00% |
| New Hires | Additive | #,0 |
| Total Terminations | Additive | #,0 |
| Net HC Change | Derived | #,0 |
| Avg Tenure Years | Average | 0.0 |
| People Managers Count | Additive | #,0 |
| IC Count | Additive | #,0 |
| Span of Control | Ratio | 0.0 |
| New Hires Cumulative | Running total | #,0 |
| Total Terminations Cumulative | Running total | #,0 |
| Net HC Change Cumulative | Running total | #,0 |

## 02 Attrition (12 + 8 cumulative)

All exit count measures use `USERELATIONSHIP(fact_attrition_v[employee_sk], dim_employee_v[employee_sk])`.

Termination type filters reference `fact_attrition_v[termination_type]` directly (not dim_employee_v) since the column lives on the fact table.

| Measure | Type | Denominator |
|---------|------|-------------|
| Total Exits | Count | — |
| Voluntary Exits | Filtered count | — |
| Involuntary Exits | Filtered count | — |
| Regrettable Exits | Filtered count | — |
| 90 Day Exits | Filtered count | — |
| Total Attrition Rate | Ratio | AVERAGEX active HC across period |
| Voluntary Attrition Rate | Ratio | AVERAGEX active HC across period |
| Regrettable Attrition Rate | Ratio | Total Exits |
| 90 Day Attrition Rate | Ratio | New Hires |
| Avg Tenure at Exit Years | Average | — |
| Attrition Rate LY | Time intelligence | — |
| Attrition Rate YoY Delta | Derived | — |

**Cumulative versions** use the REMOVEFILTERS pattern for trend line charts.

## 03 Compensation (11 measures)

| Measure | Notes |
|---------|-------|
| Avg Salary | Active employees only |
| Avg Salary LY | DATEADD -1 YEAR |
| Avg Salary YoY Change | Ratio, format 0.00% |
| Avg Compa Ratio | 1.0 = at midpoint |
| Below Band Min | compa_ratio < 0.8 |
| Above Band Max | compa_ratio > 1.2 |
| Total Payroll | Annual salary / 12, active only |
| Merit Increases Count | change_reason = "Merit" |
| Promotion Increases Count | change_reason = "Promotion" |
| Avg Merit Increase | Average change_pct for merit |
| Avg Promotion Increase | Average change_pct for promotion |

## 04 Performance (6 measures)

| Measure | Notes |
|---------|-------|
| Avg Rating | Simple average |
| High Performer Count | is_high_performer = 1 |
| High Performer Rate | Count / total reviews |
| PIP Count | is_pip = 1 |
| PIP Rate | Count / total reviews |
| Promotion Rate | Promotion count / active HC |

## 05 Workforce Planning (8 measures)

**Note:** Approved Headcount and Approved Payroll come from fact_budget_v which connects only to dim_department_v and dim_date_v. Employee attribute slicers (gender, level) do not affect budget measures.

| Measure | Notes |
|---------|-------|
| Approved Headcount | SUM from fact_budget_v |
| HC Variance | Active - Approved |
| HC Variance Pct | Variance / Approved |
| Approved Payroll | SUM from fact_budget_v |
| Actual Payroll | References Total Payroll |
| Payroll Variance | Actual - Approved |
| Payroll Variance Pct | Variance / Approved |
| HC Plan Achieved | Active / Approved |

## 06 Diversity (9 + 2 cumulative)

| Measure | Notes |
|---------|-------|
| Female HC | Filters dim_employee_v[gender] = "Female" |
| Male HC | Filters dim_employee_v[gender] = "Male" |
| Female Workforce Share | Female / Total, format 0.00% |
| Gender Pay Gap | (Male avg salary - Female avg salary) / Male avg salary |
| Female Compa Ratio | Female average compa ratio |
| Male Compa Ratio | Male average compa ratio |
| Compa Ratio Gap | Male ratio - Female ratio |
| Female Promotion Rate | Female promotions / Female HC |
| Male Promotion Rate | Male promotions / Male HC |

## 07 Executive Summary (ES measures)

ES measures pin snapshot metrics to the last complete month within the selected filter context using `FILTER(ALLSELECTED(dim_date_v[yyyymm]), dim_date_v[yyyymm] = LCM)`.

YoY comparison labels use `REMOVEFILTERS(dim_date_v)` to reach prior year data outside the current slicer selection.

Flow measures (exits, hires, net change) pass through to base measures with no date override — they accumulate across the selected period naturally.

## 08 Compensation & Equity (page-specific measures)

ES-prefixed measures specific to the Compensation & Pay Equity report page.
