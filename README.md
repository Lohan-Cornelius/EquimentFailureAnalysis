# **Equipment Failure & Root Cause Analysis — Project Log**

Author: Lohan Cornelius <br/>
Tools: MySQL → Power BI <br/>
Dataset: equipment_failure_data.csv (422 rows, simulated print-press failure records)

## **Project Overview**

This project mirrors real diagnostic work on print-press equipment, applying structured data analysis tooling (SQL, dashboarding) to a messy, realistic failure-log dataset. The dataset was intentionally seeded with common real-world data quality issues: inconsistent text casing/spacing, mixed date formats, missing values, invalid negative numbers, and inconsistent boolean encodings.

The goal is to take this raw, messy log all the way through to a working root-cause dashboard and a set of actionable findings, the same journey real machine-failure data takes on the shop floor, just with proper tooling behind it instead of a spreadsheet.

## **Workflow**
1.  Load and inspect - Import the raw CSV into MySQL untouched, and run baseline checks to understand exactly how messy the data is (row counts, duplicates, distinct     values in inconsistent columns) before deciding how to clean it.
2.  Clean with SQL, Standardize text fields (casing, whitespace, mapping to a clean lookup of real part names), normalize the resolved column into a true boolean,        flag and correct invalid negative values, parse the mixed date formats into a single consistent format, and de-duplicate records.
3.  Analyze with SQL - Query the cleaned data to answer the real business questions: which machine has the highest downtime, which parts fail most often, average         cost per failure type, seasonal patterns, and which root causes are most linked to preventable failures.
4.  Build the Power BI dashboard — Turn the SQL analysis into an interactive dashboard: KPI cards, a failures-by-part bar chart, a failures-over-time line chart,      top-5-root-causes table, and slicers to filter by machine or technician.
5.  Write it up - Summarize the findings and recommendations in plain language, as if explaining them to a non-technical stakeholder rather than a room of             analysts.
6.  Publish - Push the SQL scripts and this README to GitHub, and share the dashboard and write-up as a portfolio piece.



### **Step 1: Load and Inspect**

Goal :
Get the raw CSV into MySQL untouched, and establish a baseline understanding of its size and quality issues before touching any cleaning logic.

What I did
1.  Created a database and a raw staging table.
2.  Loaded the CSV using MySQL's bulk import.
3.  Ran baseline checks (row count, duplicate IDs, distinct values in messy columns).
