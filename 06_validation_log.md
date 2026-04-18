# 06 — Validation Log

## Validation Framework

Notebook `05_validation` runs 11 automated data quality checks. All checks must PASS before any Power BI refresh. Results are logged to `silver.reconciliation_log` with UUID run IDs and timestamps.

## Check Inventory

| # | Check Name | What It Validates | Expected | Threshold |
|---|-----------|-------------------|----------|-----------|
| 1 | Bronze row counts | All 5 Bronze tables have data | Row count > 0 for each | Any zero = FAIL |
| 2 | Silver row counts | All Silver tables populated | Row count > 0 for each | Any zero = FAIL |
| 3 | Employee count | 5,000 unique employees | emp_id distinct count = 5,000 | Exact match |
| 4 | SCD2 integrity | Every emp_id has exactly one is_current=1 row | Zero duplicates | Any duplicate = FAIL |
| 5 | Referential integrity | All fact table employee_sk values exist in dim_employee | Zero orphans | Any orphan = FAIL |
| 6 | Date range | Data spans expected fiscal year range | Dates within START_DATE to END_DATE | Any out-of-range = FAIL |
| 7 | Attrition rate range | Voluntary attrition rate is realistic | Between 5% and 25% | Outside range = FAIL |
| 8 | Compa ratio range | No extreme outliers | All between 0.5 and 2.0 | Any out-of-range = FAIL |
| 9 | Gender distribution | Realistic gender split | Female 40-55%, Male 40-55% | Outside range = FAIL |
| 10 | Pay equity gap | Engineered gender gap exists at IC4+ | Gap between 2% and 15% | Outside range = FAIL |
| 11 | Headcount swings | No unrealistic month-over-month changes | ABS(mom_hc_pct) <= 50% when prior_month_hc > 200 | Any violation = FAIL |

## CHECK 11 Threshold Decision

The original threshold was 20% MoM change. This produced false positives in early months of the data range where the employee base is small:

- December 2020 → January 2021: ~55 → ~97 employees = 76% MoM growth
- This is mathematically expected during ramp-up, not a data quality error

The threshold was adjusted to: `ABS(mom_hc_pct) > 50 AND prior_month_hc > 200`

This combination:
- Ignores ramp-up noise (small base populations can swing wildly in percentage terms)
- Still catches genuine pipeline failures on established populations (>200 employees)
- 50% MoM change on a population of 200+ would indicate a data loading issue, not organic growth

## Run Log

| Run Date | Run ID | Checks Passed | Status | Notes |
|----------|--------|---------------|--------|-------|
| 2026-04-13 | (uuid) | 11/11 | ALL PASS | Initial validation after dynamic date range update |

## How to Use

1. After any data pipeline change, run notebook `05_validation`
2. Execute Cell 18 (the UNION ALL summary) — all 11 rows must show PASS
3. If any check fails, do NOT refresh Power BI. Investigate using the diagnostic queries in each check cell
4. After confirming all PASS, run Cell 19 to log results to `silver.reconciliation_log`
5. Refresh Power BI

## Failure Remediation

| Check Fails | Likely Cause | Fix |
|-------------|-------------|-----|
| CHECK 1 | Bronze data generation failed | Re-run notebook 01 |
| CHECK 2 | Silver build failed | Re-run notebook 02 |
| CHECK 3-4 | Employee count or SCD2 issue | Re-run notebook 01, then 02 |
| CHECK 5 | Orphaned fact records | Rebuild Gold (notebook 03) after fixing Silver |
| CHECK 7 | Attrition rate out of range | Check attrition probability in data generator |
| CHECK 10 | Pay gap not showing | Verify gender_salary_adjustment function in notebook 01 |
| CHECK 11 | Large HC swing on large population | Investigate specific month: `SELECT * FROM gold.v_headcount_trend WHERE ABS(mom_hc_pct) > 50 AND prior_month_hc > 200` |
