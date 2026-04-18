# 03 — Data Dictionary

## dim_employee_v

| Column | Type | Nullable | Definition | Common Mistakes |
|--------|------|----------|------------|-----------------|
| employee_sk | INT | No | Surrogate key unique per SCD2 version. Created by ROW_NUMBER() OVER (ORDER BY emp_id, scd_start_date) | Joining on emp_id instead causes many-to-many because emp_id appears on multiple SCD2 versions |
| emp_id | STRING | No | Business key for the employee (e.g., E000001). Same across all SCD2 versions | Do not use for fact table joins — use employee_sk |
| version | INT | No | SCD2 version number. 1 = original hire record, 2+ = subsequent changes | |
| full_name | STRING | No | Employee display name | |
| gender | STRING | No | Male, Female, Non-binary, Prefer not to say | Used directly as slicer — no separate slicer table needed |
| ethnicity | STRING | No | Employee ethnicity | |
| birth_date | DATE | No | Date of birth | |
| hire_date | DATE | No | Original hire date (same across all SCD2 versions) | |
| employment_type | STRING | No | Full-time, Part-time, Contractor | |
| dept_id | STRING | No | Foreign key to dim_department_v | |
| dept_name | STRING | No | Department name (denormalized for convenience) | |
| business_unit | STRING | No | Business unit (denormalized from department) | |
| region | STRING | No | AMER, EMEA, APAC | Used directly as slicer |
| location | STRING | No | Office location | |
| job_code | STRING | No | Foreign key to dim_job_v (e.g., SWE-IC3) | |
| job_title | STRING | No | Job title | |
| job_family | STRING | No | Job family grouping | |
| job_level | STRING | No | IC1-IC5, M1-M3 | |
| is_people_manager | BOOLEAN | No | TRUE if the employee manages others | |
| salary | DECIMAL | No | Annual base salary | |
| comp_band | STRING | No | Compensation band name. Foreign key to dim_comp_band_v | |
| compa_ratio | DECIMAL | No | salary / band_midpoint. 1.0 = at midpoint. Below 0.8 = underpayment risk | Not a percentage — it's a ratio |
| manager_id | STRING | Yes | emp_id of the employee's manager. NULL for top-level | |
| status | STRING | No | Active or Terminated | |
| termination_date | DATE | Yes | NULL if still active | |
| termination_reason | STRING | Yes | Growth Opportunity, Compensation, etc. NULL if active | |
| termination_type | STRING | Yes | Voluntary, Involuntary, Retirement. NULL if active | |
| is_active_current | INT | No | Pre-computed flag: 1 = is_current=1 AND status=Active. Use this instead of filtering two columns separately | |
| is_current | INT | No | SCD2 flag. 1 = this is the latest version of this employee | |
| scd_start_date | DATE | No | When this version became effective | |
| scd_end_date | DATE | Yes | When this version was superseded. NULL if current | |
| tenure_band_today | STRING | No | < 1 Year, 1-3 Years, 3-5 Years, 5-10 Years, 10+ Years. Computed from hire_date to CURRENT_DATE() | Requires slicer_tenure_v for sort order |
| tenure_days_today | INT | No | Days since hire_date | |
| tenure_years_today | DECIMAL | No | Years since hire_date | |

## dim_date_v

| Column | Type | Definition |
|--------|------|------------|
| date_sk | INT | YYYYMMDD integer key. Join column for all fact tables |
| date | DATE | Calendar date |
| year | INT | Calendar year |
| quarter | INT | Calendar quarter 1-4 |
| month | INT | Calendar month 1-12 |
| month_name | STRING | Full month name. Sort by: month |
| fiscal_year | INT | Adobe fiscal year integer. Dec 2025 = 2026 |
| fiscal_year_label | STRING | FY2026 format |
| fiscal_quarter | INT | Adobe fiscal quarter 1-4. Q1 = Dec-Feb |
| fiscal_quarter_label | STRING | Q1, Q2, Q3, Q4. Sort by: fiscal_quarter |
| fiscal_quarter_year_label | STRING | Q1 FY2026 format |
| fiscal_month | INT | Fiscal month 1-12. Dec = 1, Nov = 12 |
| yyyymm | INT | YYYYMM integer. Used for dim_month_v bridge and trend joins |
| month_year_label | STRING | "Mar 2026" format. Sort by: fiscal_yyyyqm |

## fact_headcount_v

| Column | Type | Definition |
|--------|------|------------|
| employee_sk | INT | Foreign key to dim_employee_v. Captures the correct SCD2 version at snapshot time |
| date_sk | INT | Foreign key to dim_date_v. First day of the snapshot month (YYYYMM01) |
| is_active | INT | 1 = employee was active in this month. SUM(is_active) = active headcount |
| is_new_hire | INT | 1 = employee's hire_date falls in this month |
| is_termination | INT | 1 = employee's termination_date falls in this month |
| salary | DECIMAL | Annual salary as of this month |
| compa_ratio | DECIMAL | Compa ratio as of this month |
| tenure_years | DECIMAL | Tenure in years at the time of this snapshot |
| tenure_band | STRING | Tenure band at time of snapshot |
| is_people_manager | BOOLEAN | Manager status at time of snapshot |
| snapshot_yyyymm | INT | YYYYMM of the snapshot month |

## fact_attrition_v

| Column | Type | Definition |
|--------|------|------------|
| employee_sk | INT | Foreign key to dim_employee_v. INACTIVE relationship — activated via USERELATIONSHIP in measures |
| date_sk | INT | Foreign key to dim_date_v. ACTIVE relationship — date slicers filter directly |
| termination_type | STRING | Voluntary, Involuntary, Retirement |
| termination_reason | STRING | Growth Opportunity, Compensation, Manager/Culture, Life Event, Involuntary - PIP, Retirement, Involuntary - Restructure |
| is_regrettable | INT | 1 = the exit was regrettable (company wanted to retain) |
| is_90_day_exit | INT | 1 = employee left within 90 days of hire |
| tenure_at_exit_years | DECIMAL | Tenure at time of termination |

## fact_budget_v

| Column | Type | Definition |
|--------|------|------------|
| date_sk | INT | Foreign key to dim_date_v |
| dept_id | STRING | Foreign key to dim_department_v. Budget is planned at department level |
| approved_headcount | INT | Finance-approved headcount target for this department and month |
| approved_payroll | DECIMAL | Finance-approved payroll budget |
| fiscal_year | INT | Adobe fiscal year for this budget record |

**Note:** fact_budget_v connects to dim_department_v and dim_date_v only — it has no relationship to dim_employee_v. Employee attribute slicers (gender, job level) do not filter budget data. This is correct: budgets are planned at the department level, not the employee level.
