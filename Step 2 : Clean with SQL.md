# **Clean with SQL**

## **Goal**

## **Turn the messy raw table (equipment_failures_raw) into a clean, analysis-ready table — without ever overwriting the raw data, so the before/after transformation is always visible in the portfolio.**

## **What I did**

  1. De-duplicated records, keeping the most complete row per duplicate group.
  2. Standardized resolved into a clean boolean/num.
  3. Normalized part_affected spellings to a lookup table of real part names.
  4. Parsed date_reported from mixed formats into a single DATE type.
  5. Corrected/flagged negative downtime_hours and estimated_cost_zar values.
  6.Converted blank fields to proper NULLs and decided how to handle each.

## Walkthrough

## 1. De-duplicated records, keeping the most complete row per duplicate group.
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

### Checking Completeness Of A Row
```sql
SELECT
    record_id,
    machine_id,
    (part_affected <> '') AS has_part,
    (failure_type <> '') AS has_failure_type,
    (downtime_hours <> '') AS has_downtime,
    (estimated_cost_zar <> '') AS has_cost,
    (technician <> '') AS has_technician,
    (root_cause <> '') AS has_root_cause,
    (resolved <> '') AS has_resolved
FROM equipment_failures_raw
WHERE record_id IN (
    SELECT record_id
    FROM equipment_failures_raw
    GROUP BY record_id, machine_id, date_reported
    HAVING COUNT(*) > 1
)
ORDER BY record_id;
```

<img width="1032" height="572" alt="RS53" src="https://github.com/user-attachments/assets/258fe0d9-6f3b-4d48-8d59-3bc8660c6db2" />

Clear that Duplicate Rows are pure duplicates as the Value (True & False) query above has the same 1's & 0's across all duplicate rows. Would not matter which duplicate you delete, the data will not lose integrity.


Created New Table with Distinct Rows from the first table.
```sql
CREATE TABLE equipment_failures_dedup AS
SELECT DISTINCT *
FROM equipment_failures_raw;
```


```sql
SELECT COUNT(*) AS total_rows
FROM equipment_failures_dedup;
```
<img width="737" height="311" alt="RS56" src="https://github.com/user-attachments/assets/404c7cd7-7d2e-4909-8a71-024dc4501e93" />

Clear that my initial conclusion that all are exact duplicates was incorrect as there are 6 lines being seen as unique, irrespective of their record_id's matching.
This means that the mismatch is within the values of other columns.

```sql
SELECT *
FROM equipment_failures_dedup
WHERE record_id IN (
    SELECT record_id
    FROM equipment_failures_dedup
    GROUP BY record_id
    HAVING COUNT(*) > 1
)
ORDER BY record_id;
```
<img width="1185" height="322" alt="RS60" src="https://github.com/user-attachments/assets/a7c7d253-823b-4454-967b-92b29721ea44" />

Clear that the reason for the duplicates are due to a mismatch in the downtime_hours column.


We have 3 options :
  1. Take the Higher Value
  2. Take the Lower Value
  3. Average the Values

Decision : 
I will




## 2. Standardized resolved into a clean boolean/num.
```sql

```
