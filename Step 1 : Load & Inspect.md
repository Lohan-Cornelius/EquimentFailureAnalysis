**Goal** : Get the raw CSV into MySQL untouched, and establish a baseline understanding of its size and quality issues before touching any cleaning logic.

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

 
