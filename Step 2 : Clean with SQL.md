# **Clean with SQL**

## **Goal**

## **Turn the messy raw table (equipment_failures_raw) into a clean, analysis-ready table — without ever overwriting the raw data, so the before/after transformation is always visible in the portfolio.**

## **What I did**

  1. De-duplicated records, keeping the most complete row per duplicate group.
  2. Standardized resolved into a clean boolean.
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
I will be keeping the higher value.

Reasoning: 
Downtime is more often under-reported than over-reported in the moment, a technician may log an initial, optimistic estimate while the press is still down, then
the actual figure comes in higher once the machine is fully back in service. Keeping the higher value is the more conservative (and more operationally realistic)
choice, since underestimating downtime risks masking the true cost and urgency of a failure. The lower duplicate in each pair is treated as the earlier/less
accurate reading.


```sql
SET SQL_SAFE_UPDATES = 0;

/* Step 1: Create a deduplicated table, collapsing exact duplicate rows */
CREATE TABLE equipment_failures_dedup_none AS
SELECT DISTINCT *
FROM equipment_failures_raw;

/* Step 2: Resolve the remaining 6 record_ids with conflicting downtime_hours */
/* Decision: keep the higher value (see reasoning in project log) */
DELETE t1
FROM equipment_failures_dedup t1
JOIN equipment_failures_dedup t2
    ON t1.record_id = t2.record_id
    AND t1.downtime_hours < t2.downtime_hours;

/* Step 3: Verify — should return 400 */
SELECT COUNT(*) AS total_rows
FROM equipment_failures_dedup;
```
<img width="662" height="92" alt="RS58" src="https://github.com/user-attachments/assets/9b41712e-b3ad-4cd9-826e-bb8d80a9a3cc" />

Clear that duplicates has been removes, can move to step 2.




## 2. Standardized resolved into a clean boolean.
```sql
ALTER TABLE equipment_failures_dedup
ADD COLUMN resolved_clean BOOLEAN;
```
<img width="1296" height="266" alt="RS63" src="https://github.com/user-attachments/assets/4373eb3d-d995-42cf-91ba-c5643ebf2cdf" />


Added Column for Boolean interpretation of whether the incident was resolved or not.

### Assigning True or False dependant on the value in the resolved column, NULL if no value entired.
```sql
UPDATE equipment_failures_dedup
SET resolved_clean = CASE
    WHEN resolved IN ('Yes', 'yes', 'Y') THEN TRUE
    WHEN resolved IN ('No', 'N') THEN FALSE
    ELSE NULL
END;
```
<img width="1305" height="477" alt="RS64" src="https://github.com/user-attachments/assets/0e2a345e-aee2-47ab-b023-578588699da4" />

Updated new column to accurately portray whether is was resolved or not.

### Checking to see if assignment in resolved_clean is correct

```sql
SELECT
    resolved,
    resolved_clean,
    COUNT(*) AS row_count
FROM equipment_failures_dedup
WHERE
    (resolved IN ('Yes', 'yes', 'Y') AND resolved_clean <> TRUE)
    OR (resolved IN ('No', 'N') AND resolved_clean <> FALSE)
    OR (resolved = '' AND resolved_clean IS NOT NULL)
GROUP BY resolved, resolved_clean;
```
<img width="662" height="267" alt="RS69" src="https://github.com/user-attachments/assets/77eac70c-32e9-4323-9949-06f73469f39c" />

Can Clearly see that there are no assignments that are incorrect as I ran a check with the above query.
What this does: it defines "what should have happened" for each case, and only returns rows where the actual resolved_clean value doesn't match what was expected.
If my UPDATE logic was correct, this query would return zero rows, an empty result is actually the good outcome here, since it means no mismatches were found.

### Dropping resolved column and renaming resolved_clean

Dropping Column

```sql
ALTER TABLE equipment_failures_dedup
DROP COLUMN resolved;
```

Renaming Column

```sql
ALTER TABLE equipment_failures_dedup
CHANGE COLUMN resolved_clean resolved BOOLEAN;
```

## Normalized part_affected spellings to a lookup table of real part names.

### Selecting Distinct part_affected
```sql
SELECT DISTINCT part_affected
FROM equipment_failures_dedup
ORDER BY 1;
```
<img width="757" height="540" alt="RS71" src="https://github.com/user-attachments/assets/436ec1cf-5800-458f-aafb-408a0fa60eb7" />

- **Blanket Cylinder**	: BLANKET CYLINDER, Blanket cylinder , blanket cylinder
- **Dampening Roller** :	Damp. Roller, dampening roller
- **Delivery Chain** :	Delivery Chian, delivery chain
- **Doctor Blade** :	Dr. Blade, doctor blade
- **Feeder Unit** :	Feeder Unit, feeder unit
- **Gripper Bar** :	Gripper Bar, gripper bar
- **Impression** :	impression cyl.
- **Ink Roller** :	Ink Roller, Ink Rollr, ink roller
- **Plate Clamp** :	Plate Clamp , plate clamp
- **Registration Sensor** :	Reg. Sensor, registration sensor
- **Blank rows too**

### Creating the Lookup Table
```sql
CREATE TABLE part_lookup (
    messy_value VARCHAR(100),
    clean_value VARCHAR(100)
);
```
<img width="612" height="107" alt="image" src="https://github.com/user-attachments/assets/ca7b9b11-94be-40ef-93a2-5195c587ce47" />

### Inserting data into Lookup table
```sql
INSERT INTO part_lookup (messy_value, clean_value) VALUES
('BLANKET CYLINDER',      'Blanket Cylinder'),
('Blanket Cylinder',      'Blanket Cylinder'),
('Blanket cylinder ',     'Blanket Cylinder'),
('blanket cylinder',      'Blanket Cylinder'),

('Damp. Roller',          'Dampening Roller'),
('Dampening Roller',      'Dampening Roller'),
('dampening roller',      'Dampening Roller'),

('Delivery Chain',        'Delivery Chain'),
('Delivery Chian',        'Delivery Chain'),
('delivery chain',        'Delivery Chain'),

('Doctor Blade',          'Doctor Blade'),
('Dr. Blade',             'Doctor Blade'),
('doctor blade',          'Doctor Blade'),

('Feeder  Unit',          'Feeder Unit'),
('Feeder Unit',           'Feeder Unit'),
('feeder unit',           'Feeder Unit'),

('Gripper  Bar',          'Gripper Bar'),
('Gripper Bar',           'Gripper Bar'),
('gripper bar',           'Gripper Bar'),

('Impression Cylinder',   'Impression Cylinder'),
('impression cyl.',       'Impression Cylinder'),

('Ink  Roller',           'Ink Roller'),
('Ink Roller',            'Ink Roller'),
('Ink Rollr',             'Ink Roller'),
('ink roller',            'Ink Roller'),

('Plate Clamp',           'Plate Clamp'),
('Plate Clamp ',          'Plate Clamp'),
('plate clamp',           'Plate Clamp'),

('Reg. Sensor',           'Registration Sensor'),
('Registration Sensor',   'Registration Sensor'),
('registration sensor',   'Registration Sensor');
```

<img width="672" height="462" alt="image" src="https://github.com/user-attachments/assets/7553a5b1-c4cc-459f-bf44-7f4f5f096e63" />

### Adding new column to equipment_failure_dedup
```sql
ALTER TABLE equipment_failures_dedup
ADD COLUMN part_affected_clean VARCHAR(100);
```

### Filling new Column with data from Lookup Table
```sql
UPDATE equipment_failures_dedup e
JOIN part_lookup p
    ON TRIM(LOWER(e.part_affected)) = TRIM(LOWER(p.messy_value))
SET e.part_affected_clean = p.clean_value;
```
<img width="1377" height="527" alt="image" src="https://github.com/user-attachments/assets/4e00cc06-ed81-4db9-94e1-08e86630a37a" />

Blank rows are kept NULL

### Verifying all allocation were correct without errors
```sql
SELECT part_affected, part_affected_clean, COUNT(*) AS row_count
FROM equipment_failures_dedup
GROUP BY part_affected, part_affected_clean
ORDER BY part_affected;
```
<img width="812" height="512" alt="image" src="https://github.com/user-attachments/assets/c2db98b0-5484-4a5c-9b90-0902fcce5e71" />

Visual check to see if the wrong original part_affected column is mapping to the correct cleaned part_affected column

- All is correct.

  ### Dropping original pat_affected column and replacing with part_affected_clean (Renamed to part_affedtec)
```sql
ALTER TABLE equipment_failures_dedup
DROP COLUMN part_affected;

ALTER TABLE equipment_failures_dedup
CHANGE COLUMN part_affected_clean part_affected VARCHAR(100);
```
<img width="1262" height="615" alt="image" src="https://github.com/user-attachments/assets/3f7fe5cf-45f3-422b-a91c-fa245b86750b" />

## 4. Parsed date_reported from mixed formats into a single DATE type.

### Adding new column to hold parsed date data
```sql
ALTER TABLE equipment_failures_dedup
ADD COLUMN date_reported_clean DATE;
```

### Cleaning date_reported into the date_reported_clean
```sql
UPDATE equipment_failures_dedup
SET date_reported_clean = CASE
    WHEN date_reported REGEXP '^[0-9]{4}-[0-9]{2}-[0-9]{2}$'
        THEN STR_TO_DATE(date_reported, '%Y-%m-%d')
    WHEN date_reported REGEXP '^[0-9]{2}/[0-9]{2}/[0-9]{4}$'
        THEN STR_TO_DATE(date_reported, '%d/%m/%Y')
    WHEN date_reported REGEXP '^[0-9]{2}-[A-Za-z]{3}-[0-9]{4}$'
        THEN STR_TO_DATE(date_reported, '%d-%b-%Y')
    ELSE NULL
END;
```
<img width="1312" height="512" alt="image" src="https://github.com/user-attachments/assets/d9af656f-b327-4a7f-955b-25f756a50bcf" />

Verify 
```sql
SELECT date_reported, date_reported_clean
FROM equipment_failures_dedup
WHERE date_reported_clean IS NULL;
```

### Correct Table
```sql
ALTER TABLE equipment_failures_dedup
DROP COLUMN date_reported;

ALTER TABLE equipment_failures_dedup
CHANGE COLUMN date_reported_clean date_reported DATE
AFTER part_affected;
```
<img width="1211" height="497" alt="image" src="https://github.com/user-attachments/assets/54eae8bb-d922-4e62-84a2-4cf1cbdb18cb" />

## Corrected negative downtime_hours and estimated_cost_zar values. 6.Converted blank fields to proper NULLs and decided how to handle each.
```sql
-- Convert blank ('') values to true NULLs so the columns can be
-- safely converted to a numeric type in the next step
UPDATE equipment_failures_dedup
SET downtime_hours = NULL
WHERE TRIM(downtime_hours) = '';

UPDATE equipment_failures_dedup
SET estimated_cost_zar = NULL
WHERE TRIM(estimated_cost_zar) = '';
```
Converted Blank values to proper NULLs.

```sql
-- Convert downtime_hours and estimated_cost_zar from text to proper decimal numbers
ALTER TABLE equipment_failures_dedup
MODIFY COLUMN downtime_hours DECIMAL(10,2),
MODIFY COLUMN estimated_cost_zar DECIMAL(10,2);
```
Converted downtime_hours and estimated_cost_zar from text to proper decimal numbers.
<img width="1187" height="495" alt="image" src="https://github.com/user-attachments/assets/06f35d5d-d6d2-4e0f-993b-dbee098b7050" />

```sql
-- Correct negative downtime_hours and estimated_cost_zar values by flipping
-- their sign, treating the negative as a data-entry sign error
UPDATE equipment_failures_dedup
SET downtime_hours = ABS(downtime_hours),
    estimated_cost_zar = ABS(estimated_cost_zar);
```
<img width="1186" height="490" alt="image" src="https://github.com/user-attachments/assets/58ad6351-1cd3-4693-83cd-0da22ed2ca81" />


Correcting negative downtime_hours and estimated_cost_zar values flipping from - to +, treating as a data-entry error.
