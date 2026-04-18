# 02 — View Inventory and Dependencies

## Semantic Layer: 16 Views

### Dimension Views (6)

| View | Source | Grain | Rows | Key Columns | Power BI Pages |
|------|--------|-------|------|-------------|----------------|
| `dim_employee_v` | silver.dim_employee | One row per SCD2 version | ~8,961 | employee_sk, emp_id, gender, job_level, dept_id, is_active_current, tenure_band_today | All pages |
| `dim_date_v` | silver.dim_date | One row per day | ~1,960 | date_sk, date, yyyymm, fiscal_year, fiscal_year_label, fiscal_quarter_label, month_year_label | All pages |
| `dim_month_v` | silver.dim_date (derived) | One row per month | ~144 | yyyymm | Trend charts (bridge) |
| `dim_department_v` | silver.dim_department | One row per department | 18 | dept_id, dept_name, business_unit, region, cost_centre | Exec Summary, Workforce Planning |
| `dim_job_v` | silver.dim_job | One row per job code | 23 | job_code, job_title, job_family, job_level, job_level_sort | Compensation, Performance |
| `dim_comp_band_v` | silver.dim_comp_band | One row per band | 7 | band_name, band_min, band_mid, band_max, band_sort | Compensation |

### Fact Views (5)

| View | Source | Grain | Rows | Key Columns | Power BI Pages |
|------|--------|-------|------|-------------|----------------|
| `fact_headcount_v` | gold.fact_headcount_snapshot | Employee × month | ~99,000+ | employee_sk, date_sk, is_active, is_new_hire, is_termination, salary, compa_ratio | All pages |
| `fact_attrition_v` | gold.fact_attrition_event | One row per exit | ~770+ | employee_sk, date_sk, termination_type, termination_reason, is_regrettable, is_90_day_exit | Exec Summary, Attrition |
| `fact_compensation_v` | gold.fact_compensation_history | One row per salary change | ~6,700+ | employee_sk, date_sk, change_amount, change_pct, change_reason | Compensation |
| `fact_performance_v` | gold.fact_performance_review | One row per review | ~9,800+ | employee_sk, date_sk, rating, is_high_performer, is_pip | Performance |
| `fact_budget_v` | gold.fact_budget (via bronze) | Department × month | ~1,170 | date_sk, dept_id, approved_headcount, approved_payroll, fiscal_year | Workforce Planning |

### Trend Views (3)

| View | Source | Grain | Rows | Key Columns | Notes |
|------|--------|-------|------|-------------|-------|
| `trend_headcount_v` | gold.v_headcount_trend | One row per month | ~65 | snapshot_yyyymm, active_headcount, mom_hc_delta, yoy_hc_pct | Pre-computed MoM/YoY |
| `trend_attrition_v` | gold.v_attrition_trend | One row per month | ~65 | snapshot_yyyymm, voluntary_exits, rolling_3m_attrition | Pre-computed rolling averages |
| `trend_compensation_v` | gold.v_compensation_trend | Month × comp band | ~455 | snapshot_yyyymm, comp_band, avg_salary | Pre-computed by band |

### Slicer Views (2)

| View | Source | Rows | Sort Column | Connected To |
|------|--------|------|-------------|--------------|
| `slicer_tenure_v` | Derived from dim_employee | 5 | tenure_sort | dim_employee_v[tenure_band_today] |
| `slicer_exit_reason_v` | Derived from fact_attrition | 7 | (alphabetical) | fact_attrition_v[termination_reason] |

## Performance Notes

- Semantic views are zero-storage pass-through views — no data duplication cost
- Trend views (65-455 rows) are pre-aggregated at refresh time so line charts don't query the 99K+ row fact table
- Trend views do NOT respond to employee attribute slicers (gender, job level, etc.) — they connect through dim_month_v, not dim_employee_v
- For pages with slicers, use DAX cumulative measures against fact tables instead of trend views

## Dynamic Date Filtering

`dim_date_v` includes a WHERE clause that limits dates to the last 5 Adobe fiscal years through today. This filter evaluates at query time (CURRENT_DATE()), so the view automatically adjusts each day without manual intervention.

## Model Evolution

The original design had 21 tables with 8 disconnected slicer tables using the IN VALUES DAX pattern. This was refactored to 16 tables with relationship-based filtering after evaluating the trade-offs between DAX complexity and model simplicity. The remaining 2 slicer tables exist only where a sort column or unique relationship path is required.
