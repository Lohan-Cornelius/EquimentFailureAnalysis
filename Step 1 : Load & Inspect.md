# Step 1 : Load & Inspect

## Goal

## Get the raw CSV into MySQL untouched, and establish a baseline understanding of its size and quality issues before touching any cleaning logic.

## **What I did**
1.  Created a database and a raw staging table.
2.  Loaded the CSV using MySQL's bulk import.
3.  Ran baseline checks (row count, duplicate IDs, distinct values in messy columns).

## **Walkthrough**

### 1.  Created a database and a RAW staging Table.
 -  Note : Every column is a text/VARCHAR at this stage on purpose, casting to proper numeric/date types in Step 2 to avoid import silently failing.
 -  Code : 

```sql
CREATE DATABASE equipment_failures;
 USE equipment_failures;

CREATE TABLE equipment_failures_raw (
    record_id           VARCHAR(20),
    machine_id          VARCHAR(20),
    part_affected        VARCHAR(100),
    failure_type         VARCHAR(100),
    date_reported         VARCHAR(20),
    downtime_hours        VARCHAR(20),
    estimated_cost_zar     VARCHAR(20),
    technician           VARCHAR(100),
    root_cause           VARCHAR(200),
    resolved             VARCHAR(10)
);
```


### 2. Loaded the CSV using MySQL's bulk Import Wizard
   
   <img width="1190" height="762" alt="Failures RAW" src="https://github.com/user-attachments/assets/aa7ce4f0-6a80-45d2-8ecf-8a5ce0743256" />
   

### 3. Ran baseline checks (Row Count, Duplicate ID's, distinct values in Messy Columns.

   ### Evaluate Duplicate Entries
```sql
/*To evaluate if there are any duplicate entries*/
SELECT 
	COUNT(*)AS Total_rows,
    COUNT(DISTINCT (record_id)) AS Unique_Incidents
FROM
	equipment_failures_raw
;
```
<img width="612" height="155" alt="RS12" src="https://github.com/user-attachments/assets/b6e9231e-8e75-414e-901a-29e33f54b2d2" />

Finding : Duplicate rows as there are more rows as unique record ID's.
	   	  22 Duplicates to Resolve.

   ### Checking Resolved Values
```sql
/*Selecting distinct Resolved values*/
SELECT
	DISTINCT(resolved)
FROM
	equipment_failures_raw
;
```
<img width="612" height="167" alt="RS13" src="https://github.com/user-attachments/assets/4c29cf45-dc77-4aeb-9ce4-bc8c04e2e376" />

Finding : There is a mismatch in the categorical values of whether or not a incident was resolved or not.
		  To clean this data to be simply yes or no.
		  6 variants for what should be a 2 value field.

   ### Checking Parts Column
```sql
/*Selecting all distinct parts affected*/
SELECT
	DISTINCT(part_affected)
FROM 
	equipment_failures_raw
ORDER BY
	1
;
```
<img width="627" height="542" alt="RS14" src="https://github.com/user-attachments/assets/9e22f4f4-047b-45ed-a2a5-efb58e59a40a" />

Finding : Clear data mismatch, multiple entries of the same part with different spelling and errors.
		  To normalize this data.
		  32 Distinct spelling for what should be 10 unique parts in total.

   ### Checking Blank/Missing Values Across Table
```sql
/*Checking NULL values in all Columns*/
SELECT 
	SUM(record_id = '') AS record_id_NULLS,
    SUM(machine_id = '') AS machine_id_NULLS,
	SUM(part_affected = '') AS part_affected_NULLS ,
    SUM(failure_type = '') AS failure_type_NULLS,
    SUM(date_reported = '') AS date_reported_NULLS,
    SUM(downtime_hours = '') AS downtime_hours_NULLS,
    SUM(estimated_cost_zar = '') AS estimated_cost_zar_NULLS,
    SUM(technician = '') AS technician_NULLS,
    SUM(root_cause = '') AS root_cause_NULLS,
    SUM(resolved = '') AS resolved_NULLS
FROM 
	equipment_failures_raw 
;
```
<img width="1457" height="122" alt="RS35" src="https://github.com/user-attachments/assets/1c73832f-abcb-4845-9d55-a7031720a802" />

Findings : Checking Null values count across all columns.
		   To fix this in step 2.

   ### Checking Date Column Format
```sql
/*Checking date column for mismatches*/
SELECT 
	DISTINCT(date_reported)
FROM 
	equipment_failures_raw 
;
```
<img width="650" height="237" alt="RS36" src="https://github.com/user-attachments/assets/cec715e5-8358-4b27-8b6c-571b17103567" />

Findings : Clear Date Format Mismatch.
		   To fix in Step 2.

   ### Checking for Negative Values Where Not Possible
```sql
/*Checking for negative values across downtime and cost columns*/
SELECT 
	SUM(downtime_hours < 0) AS negative_downtime,
    SUM(estimated_cost_zar < 0) AS negative_cost
FROM 
	equipment_failures_raw 
;
```
<img width="695" height="95" alt="RS43" src="https://github.com/user-attachments/assets/f84c0f91-7211-4445-8a1d-52a992d35fb4" />

Findings : Clear indication of data captured incorrectly, impossible to have negative downtime with this data.
		   To fix in Step 2.

## Decisions Made
	1. 22 duplicate records need to be resolved in Step 2 (de-dupe on record_id + machine_id + date_reported, keeping the most complete row).
	   resolved will be standardized to a true boolean (Y/Yes → TRUE, N/No → FALSE, blank → NULL).
	2. part_affected will be cleaned via TRIM/LOWER and mapped to a lookup table of the ~10 real part names.
	3. Blank values across part_affected, downtime_hours, estimated_cost_zar, technician, root_cause, and resolved will be handled explicitly (converted to NULL and decided case-by-case whether to impute, exclude, or leave as missing for analysis).
	4. date_reported will be parsed from mixed formats into a single standard DATE type.
	5. The 14 negative rows in each of downtime_hours and estimated_cost_zar will be flagged and corrected (likely ABS() if they're a sign-flip data-entry error) or excluded if unrecoverable.
