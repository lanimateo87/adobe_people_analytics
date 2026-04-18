# 01 — Architecture Overview

## Problem Statement

The People Analytics team at Adobe manages 20+ Power BI dashboards connected to SAP HANA. During the migration to Databricks, Data Engineering frequently changes source table schemas — renaming columns, splitting tables, adding fields. Each change breaks downstream dashboards, requiring the senior BI developer to spend days tracing and fixing report-level references across all 20+ reports.

## Solution: Semantic Contract Layer

Power BI connects exclusively to views in the `semantic` schema. These views act as contracts — their column names and data types never change without coordinated notification. When Data Engineering modifies Silver tables, the semantic view absorbs the change by updating an alias. Zero dashboards break.

## Layer Architecture

| Layer | Schema | Owner | Purpose | Refresh |
|-------|--------|-------|---------|---------|
| Bronze | `people_analytics.bronze` | Data Engineering | Raw source data, landed as-is | On extract |
| Silver | `people_analytics.silver` | Data Engineering | Cleaned, typed, SCD2 surrogate keys | On extract |
| Gold | `people_analytics.gold` | People Analytics | Fact tables, pre-aggregated trends | On Silver refresh |
| Semantic | `people_analytics.semantic` | People Analytics | Contract views — Power BI's only source | On Gold refresh |

## Ownership Boundary

The handover point is between Silver and Gold. Data Engineering delivers clean Silver tables with documented schemas. People Analytics owns everything from Gold onward — transformations, business logic, semantic contracts, and Power BI reports.

This boundary means:
- DE can rename `org_unit` to `business_unit` in Silver → People Analytics updates one alias in the semantic view → Power BI sees no change
- DE can split one table into two → People Analytics updates the semantic view to JOIN them → Power BI sees the same flat structure
- DE can add new columns → they're invisible to Power BI until People Analytics explicitly exposes them through the semantic view

## Infrastructure

| Component | Value |
|-----------|-------|
| Azure Resource Group | portfolio-dev-rg (East US 2) |
| Storage Account | portfoliodevst (ADLS Gen2, hierarchical namespace) |
| Container | delta-lake |
| Databricks Workspace | portfolio-dbw-dev (Premium Serverless) |
| SQL Warehouse | portfolio-wh-dev (Serverless, X-Small, 5-min auto-stop) |
| Unity Catalog | people_analytics catalog with managed locations per schema |
| Power BI | Import mode, daily refresh via PAT token |

## The Change Rule

Column names in `semantic.*` views are contracts. They never change without notifying all dashboard owners. This is the single most important governance principle in the architecture.

## Adobe Fiscal Year Alignment

All date dimensions and measures align to Adobe's fiscal calendar:
- Fiscal year starts in **December** (December 2025 = FY2026)
- Fiscal quarters: Q1 = Dec-Feb, Q2 = Mar-May, Q3 = Jun-Aug, Q4 = Sep-Nov
- `dim_date_v` dynamically filters to the last 5 fiscal years through today's date
