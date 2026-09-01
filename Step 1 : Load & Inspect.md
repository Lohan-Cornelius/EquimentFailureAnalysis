# **Goal** : Get the raw CSV into MySQL untouched, and establish a baseline understanding of its size and quality issues before touching any cleaning logic.

**What I did**
1.  Created a database and a raw staging table.
2.  Loaded the CSV using MySQL's bulk import.
3.  Ran baseline checks (row count, duplicate IDs, distinct values in messy columns).

**Walkthrough**

1.  Created a database and a RAW staging Table.
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

2. Loaded the CSV using MySQL's bulk Import Wizard
   
   <img width="1190" height="762" alt="Failures RAW" src="https://github.com/user-attachments/assets/aa7ce4f0-6a80-45d2-8ecf-8a5ce0743256" />

3. Ran baseline checks (Row Count, Duplicate ID's, distinct values in Messy Columns.
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
		  To clean this clean this data to be simply yes or no.


