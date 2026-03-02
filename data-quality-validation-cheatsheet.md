# Data Quality & Validation Cheatsheet

Reliable data systems include validation by default.
Below are essential validation patterns for reliable data pipelines.
Goal: ensure data is accurate, consistent, and trustworthy before reaching BI.


## Core Principle

If data cannot be trusted, nothing built on it matters.


## 1) No Missing Data

### Null Checks

```sql
SELECT COUNT(*) AS null_keys
FROM FactVisits
WHERE patient_sk IS NULL;
```

Critical fields should never be NULL:

    Primary keys

    Foreign keys

    Dates

    Required metrics
    

## 2) Duplicate Detection
Duplicate Business Keys

```sql
SELECT patient_id, COUNT(*)
FROM DimPatient
GROUP BY patient_id
HAVING COUNT(*) > 1;
```

Fact Deduplication Pattern

```sql
WITH x AS (
  SELECT *,
         ROW_NUMBER() OVER (
           PARTITION BY visit_id
           ORDER BY update_ts DESC
         ) AS rn
  FROM staging
)
SELECT * FROM x WHERE rn = 1;
```

## 3) Referential Integrity

Ensure fact keys exist in dimension.

```sql
SELECT COUNT(*)
FROM FactVisits f
LEFT JOIN DimPatient d
  ON f.patient_sk = d.patient_sk
WHERE d.patient_sk IS NULL;
```

## 4) Range & Validity Checks
Value Range Validation

```sql
SELECT COUNT(*)
FROM FactVisits
WHERE length_of_stay < 0
   OR length_of_stay > 365;
```

Date Logic

```sql
WHERE discharge_date < admission_date
```

## 5) Consistency Checks

Cross-field logic validation.

Example:

```sql
status = 'Discharged'
AND discharge_date IS NULL
```

## 6) Row Count Reconciliation

Validate loads between layers.

```sql
SELECT COUNT(*) FROM staging;
SELECT COUNT(*) FROM warehouse;
```

Track differences between:

    Source

    Staging

    Final table

## 7) Freshness Checks

Ensure data is recent.

```sql
SELECT MAX(load_date)
FROM FactVisits;
```

Alert if older than expected.

## 8) Idempotency Validation

Re-running pipeline should not duplicate rows.

```sql
SELECT COUNT(*)
FROM FactVisits
GROUP BY business_key
HAVING COUNT(*) > 1;
```

## 9) Monitoring Metadata Table

Minimum tracking table:

```sql
pipeline_name
run_timestamp
rows_loaded
status
error_message
```

Used for:

    Auditing

    Troubleshooting

    SLA tracking


## Quick Validation Checklist

- ✔ Critical fields not null
- ✔ No duplicate business keys
- ✔ Fact keys exist in dimensions
- ✔ Values within valid range
- ✔ Row counts reconciled
- ✔ Latest data loaded




