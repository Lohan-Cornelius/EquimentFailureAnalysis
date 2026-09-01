# **Clean with SQL**

## **Goal**

## **Turn the messy raw table (equipment_failures_raw) into a clean, analysis-ready table — without ever overwriting the raw data, so the before/after transformation is always visible in the portfolio.**

## **What I did**

  1. De-duplicated records, keeping the most complete row per duplicate group.
  2. Standardized resolved into a clean boolean/enum.
  3. Normalized part_affected spellings to a lookup table of real part names.
  4. Parsed date_reported from mixed formats into a single DATE type.
  5. Corrected/flagged negative downtime_hours and estimated_cost_zar values.
  6.Converted blank fields to proper NULLs and decided how to handle each.

## Walkthrough

### 1. De-duplicated records, keeping the most complete row per duplicate group.
```sql
/* See the duplicate groups and how many copies of each */
SELECT
    record_id,
    machine_id,
    date_reported,
    COUNT(*) AS copies
FROM equipment_failures_raw
GROUP BY record_id, machine_id, date_reported
HAVING COUNT(*) > 1
ORDER BY copies DESC;
```
<img width="705" height="540" alt="RS45" src="https://github.com/user-attachments/assets/301d49cb-7439-4897-8947-306399426d2d" />

Result is accurate as there are 22 rows that are duplicates and the result of the query has 22 rows all with 2 'copies' meaning that there is 1 extra of each.
