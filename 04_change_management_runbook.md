# 04 — Change Management Runbook

## Scenario 1: Data Engineering Renames a Column

**Example:** DE renames `org_unit` to `business_unit` in `silver.dim_employee`.

**Steps:**
1. Open `04_build_semantic_layer` notebook (or the corresponding SQL file in the repo)
2. Find the `dim_employee_v` view definition
3. Update the column alias: change `org_unit AS business_unit` to `business_unit` (or keep the alias if the downstream name must stay the same)
4. Run the updated CREATE OR REPLACE VIEW statement in SQL Editor (cheaper than running the notebook)
5. Copy the updated SQL back to the notebook and commit to GitHub
6. Run validation notebook Cell 18 — confirm all 11 checks PASS
7. Refresh Power BI — verify no visuals show errors
8. No DAX changes needed — Power BI sees the same column name

**Time to fix:** 10 minutes. Zero dashboard impact.

## Scenario 2: Data Engineering Splits One Table Into Two

**Example:** DE splits `silver.dim_employee` into `silver.dim_employee_core` and `silver.dim_employee_contact`.

**Steps:**
1. Update the semantic view to JOIN the two new tables:
```sql
CREATE OR REPLACE VIEW semantic.dim_employee_v AS
SELECT
    c.employee_sk, c.emp_id, c.full_name, c.gender,
    ct.email, ct.phone, ct.location
FROM silver.dim_employee_core c
JOIN silver.dim_employee_contact ct ON c.emp_id = ct.emp_id
```
2. The semantic view presents the same flat structure — Power BI sees no change
3. Run validation, refresh Power BI, commit to GitHub

**Time to fix:** 15 minutes. Zero dashboard impact.

## Scenario 3: Add a New Metric

**Example:** Business requests "Involuntary Attrition Rate" as a new KPI.

**Steps:**
1. Check if the source data exists — `fact_attrition_v` already has `termination_type` with "Involuntary" values
2. No Gold or Semantic changes needed — the data is already there
3. In Tabular Editor, add the new measure:
```dax
Involuntary Attrition Rate = 
VAR InvolExits = [Involuntary Exits]
VAR AvgHC = AVERAGEX(VALUES(dim_date_v[yyyymm]), CALCULATE(SUM(fact_headcount_v[is_active])))
RETURN DIVIDE(InvolExits, AvgHC, 0)
```
4. Set Display Folder = "02 Attrition", FormatString = "0.00%"
5. Add to the relevant report page
6. Ctrl+S to save to Power BI model

**Time to add:** 5 minutes.

## Scenario 4: NULL Values Appear in a Previously Non-Nullable Column

**Example:** After a source system update, `salary` in Silver starts containing NULLs.

**Steps:**
1. Run diagnostic in SQL Editor:
```sql
SELECT COUNT(*) AS total, 
       COUNT(salary) AS non_null, 
       COUNT(*) - COUNT(salary) AS null_count
FROM silver.dim_employee
WHERE is_current = 1
```
2. If null_count > 0, investigate the source pipeline with Data Engineering
3. As a temporary fix, update the semantic view to handle NULLs:
```sql
COALESCE(salary, 0) AS salary
```
4. Add a validation check for NULL rate monitoring
5. Document the issue and remediation in the validation log

## Scenario 5: Add a New Slicer Attribute

**Example:** Business requests filtering by `employment_type` (Full-time, Part-time, Contractor).

**Steps:**
1. Check if the attribute exists on `dim_employee_v` — it does (`employment_type` column)
2. In Power BI, add a slicer visual using `dim_employee_v[employment_type]` directly
3. No new slicer table needed — the attribute has few values (3) that sort correctly alphabetically
4. Test: verify all measures on the page respond to the slicer selection
5. If a sort column is needed (values don't sort correctly), create a `slicer_employment_type_v` view with a sort column

**When a dedicated slicer table IS needed:**
- The attribute values don't sort alphabetically in the desired order (e.g., tenure bands: "< 1 Year" must come before "1-3 Years")
- The attribute exists on a fact table with no dimension relationship path (e.g., termination_reason on fact_attrition_v)

## Scenario 6: Upstream Datatype Change

**Example:** DE changes `salary` from DECIMAL(10,2) to DOUBLE in Silver.

**Steps:**
1. Update the semantic view to CAST explicitly:
```sql
CAST(salary AS DECIMAL(10,2)) AS salary
```
2. This ensures Power BI always receives the expected type regardless of upstream changes
3. Run validation to confirm no rounding issues
4. Document the type assertion in the view comments

## Scenario 7: New Department or Business Unit Added

**Example:** Adobe creates a new "AI Research" department.

**Steps:**
1. No semantic layer changes needed — `dim_department_v` is a pass-through view from Silver
2. The new department appears automatically on next data refresh
3. All slicers using `dim_department_v[dept_name]` and `dim_department_v[business_unit]` pick it up
4. Budget data for the new department will show in fact_budget_v once Finance creates the budget record
5. Verify: new department appears in all relevant slicers after refresh

## General Rules

1. **Never rename a semantic view column without checking downstream.** Every column name is a contract. Search the DAX measures library for any reference to the column before renaming.
2. **Test in SQL Editor, commit to notebook.** Use SQL Editor for cheap execution, then update the notebook and push to GitHub.
3. **Always run validation after any change.** All 11 checks must PASS before refreshing Power BI.
4. **Document every change.** Update the validation log with what changed, why, and verification results.
