# SQL Cheatsheet

## 1) Query Skeletons

### 1.1 Base query pattern
```sql
SELECT
  t.col1,
  t.col2,
  ...
FROM dbo.Table AS t
WHERE t.some_flag = 1
  AND t.event_date >= '2026-01-01'
ORDER BY t.event_date DESC;
```

### 1.2 Aggregate pattern (BI metric)

```sql
SELECT
  t.department_id,
  COUNT(*)                  AS n,
  SUM(t.amount)             AS total_amount,
  AVG(CAST(t.amount AS DECIMAL(18,2))) AS avg_amount
FROM dbo.FactEvents AS t
WHERE t.event_date >= '2026-01-01'
GROUP BY t.department_id;
```

## 2) Joins

###  2.1 Join with explicit keys + select list

```sql
SELECT
  f.visit_id,
  f.visit_date,
  p.patient_sk,
  d.department_name
FROM dbo.FactVisit AS f
LEFT JOIN dbo.DimPatient AS p
  ON f.patient_sk = p.patient_sk
LEFT JOIN dbo.DimDepartment AS d
  ON f.department_sk = d.department_sk;
```

### 2.2 Detect join explosions (many-to-many mistake)

```sql
SELECT
  COUNT(*) AS rows_after_join
FROM dbo.FactVisit f
JOIN dbo.SomeDim x
  ON f.key = x.key;
```

```sql
-- Compare to base count:
SELECT COUNT(*) AS base_rows FROM dbo.FactVisit;
```

### 2.3 Semi-join (exists) — often faster than join

```sql
SELECT f.*
FROM dbo.FactVisit f
WHERE EXISTS (
  SELECT 1
  FROM dbo.DimDepartment d
  WHERE d.department_sk = f.department_sk
    AND d.is_active = 1
);
```

### 2.4 Anti-join (find missing dimension matches)

```sql
SELECT f.department_sk, COUNT(*) AS cnt
FROM dbo.FactVisit f
LEFT JOIN dbo.DimDepartment d
  ON f.department_sk = d.department_sk
WHERE d.department_sk IS NULL
GROUP BY f.department_sk;
```

## 3) Window Functions (Advanced BI Superpower)

### 3.1 Row number / latest record per group

```sql
WITH ranked AS (
  SELECT
    e.*,
    ROW_NUMBER() OVER (
      PARTITION BY e.patient_id
      ORDER BY e.event_ts DESC
    ) AS rn
  FROM dbo.Events e
)
SELECT *
FROM ranked
WHERE rn = 1;
```

### 3.2 Running total

```sql
SELECT
  event_date,
  amount,
  SUM(amount) OVER (
    ORDER BY event_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM dbo.Payments;
```

### 3.3 Moving average (7-day)

```sql
SELECT
  event_date,
  metric,
  AVG(metric) OVER (
    ORDER BY event_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS avg_7d
FROM dbo.DailyMetrics;
```

### 3.4 Share of total (no extra join)

```sql
SELECT
  department_id,
  SUM(amount) AS dept_amount,
  SUM(amount) * 1.0 / SUM(SUM(amount)) OVER () AS pct_of_total
FROM dbo.FactSpend
GROUP BY department_id;
```

### 3.5 Top-N per group

```sql
WITH x AS (
  SELECT
    department_id,
    provider_id,
    SUM(amount) AS total_amount,
    DENSE_RANK() OVER (
      PARTITION BY department_id
      ORDER BY SUM(amount) DESC
    ) AS rnk
  FROM dbo.FactSpend
  GROUP BY department_id, provider_id
)
SELECT *
FROM x
WHERE rnk <= 5;
```

## 4) Time Series Patterns (Warehouse)

### 4.1 Date spine (SQL Server) — ensures no missing dates

```sql
WITH DateSpine AS (
  SELECT CAST('2026-01-01' AS date) AS d
  UNION ALL
  SELECT DATEADD(day, 1, d)
  FROM DateSpine
  WHERE d < '2026-01-31'
)
SELECT ds.d, COALESCE(m.cnt, 0) AS cnt
FROM DateSpine ds
LEFT JOIN (
  SELECT CAST(event_date AS date) AS d, COUNT(*) AS cnt
  FROM dbo.FactEvents
  GROUP BY CAST(event_date AS date)
) m ON m.d = ds.d
ORDER BY ds.d
OPTION (MAXRECURSION 1000);
```

### 4.2 Period-to-date (Month-to-date example)

```sql
SELECT
  event_date,
  amount,
  SUM(amount) OVER (
    PARTITION BY YEAR(event_date), MONTH(event_date)
    ORDER BY event_date
  ) AS mtd
FROM dbo.FactSpend;
```

## 5) Data Cleaning / Standardization

### 5.1 CASE for mapping categories

```sql
SELECT
  *,
  CASE
    WHEN status IN ('A','ACTIVE') THEN 'Active'
    WHEN status IN ('I','INACTIVE') THEN 'Inactive'
    ELSE 'Unknown'
  END AS status_std
FROM dbo.Users;
```

## 5.2 COALESCE / NULLIF

```sql
SELECT
  COALESCE(phone, mobile, 'N/A') AS best_phone,
  amount / NULLIF(quantity, 0)   AS unit_price
FROM dbo.Orders;
```

## 5.3 Robust casting (SQL Server)
```sql
SELECT
  TRY_CONVERT(date, date_str) AS parsed_date,
  TRY_CONVERT(int,  num_str)  AS parsed_int
FROM dbo.StageInput;
```

## 6) CTEs, Temp Tables, and Modular Queries

### 6.1 Multi-CTE pipeline style

```sql
WITH base AS (
  SELECT *
  FROM dbo.StageEvents
  WHERE load_batch_id = @batch_id
),
clean AS (
  SELECT
    event_id,
    TRY_CONVERT(datetime2, event_ts) AS event_ts,
    NULLIF(TRIM(event_type), '')     AS event_type
  FROM base
),
dedup AS (
  SELECT *
  FROM (
    SELECT
      *,
      ROW_NUMBER() OVER (PARTITION BY event_id ORDER BY event_ts DESC) AS rn
    FROM clean
  ) x
  WHERE rn = 1
)
SELECT * FROM dedup;
```

### 6.2 Temp table (good for complex transformations)

```sql
SELECT ...
INTO #stg
FROM dbo.StageEvents
WHERE load_batch_id = @batch_id;
```

```sql
CREATE INDEX IX_stg_event_id ON #stg(event_id);
```

## 7) Upserts / MERGE (ETL Essentials)

### 7.1 MERGE into dimension (Type 1 overwrite)

```sql
MERGE dbo.DimDepartment AS tgt
USING dbo.StageDepartment AS src
  ON tgt.department_id = src.department_id
WHEN MATCHED THEN
  UPDATE SET
    tgt.department_name = src.department_name,
    tgt.updated_at = SYSUTCDATETIME()
WHEN NOT MATCHED THEN
  INSERT (department_id, department_name, created_at, updated_at)
  VALUES (src.department_id, src.department_name, SYSUTCDATETIME(), SYSUTCDATETIME());
```

### 7.2 SCD Type 2 (history tracking) — simplified pattern

```sql
-- Assumes: Dim has (business_key, attr..., valid_from, valid_to, is_current)
-- Step 1: expire changed current rows
UPDATE d
SET
  d.valid_to = SYSUTCDATETIME(),
  d.is_current = 0
FROM dbo.DimPatient d
JOIN dbo.StagePatient s
  ON d.patient_id = s.patient_id
WHERE d.is_current = 1
  AND (
    ISNULL(d.last_name,'') <> ISNULL(s.last_name,'')
    OR ISNULL(d.postal_code,'') <> ISNULL(s.postal_code,'')
  );
```

```sql
-- Step 2: insert new current versions (new or changed)
INSERT INTO dbo.DimPatient (patient_id, last_name, postal_code, valid_from, valid_to, is_current)
SELECT
  s.patient_id, s.last_name, s.postal_code,
  SYSUTCDATETIME(), '9999-12-31', 1
FROM dbo.StagePatient s
LEFT JOIN dbo.DimPatient d
  ON d.patient_id = s.patient_id AND d.is_current = 1
WHERE d.patient_id IS NULL
   OR (
     ISNULL(d.last_name,'') <> ISNULL(s.last_name,'')
     OR ISNULL(d.postal_code,'') <> ISNULL(s.postal_code,'')
   );
```

## 8) Reconciliation & Data Quality (Stand-out DE habit)

### 8.1 Row count reconciliation

```sql
SELECT 'stage' AS layer, COUNT(*) AS cnt FROM dbo.StageEvents WHERE load_batch_id = @batch_id
UNION ALL
SELECT 'fact'  AS layer, COUNT(*) AS cnt FROM dbo.FactEvents  WHERE load_batch_id = @batch_id;
```

### 8.2 Duplicate key check

```sql
SELECT business_key, COUNT(*) AS cnt
FROM dbo.DimPatient
WHERE is_current = 1
GROUP BY business_key
HAVING COUNT(*) > 1;
```

### 8.3 Null critical fields

```sql
SELECT COUNT(*) AS null_patient
FROM dbo.FactVisit
WHERE patient_sk IS NULL;
```

## 9) Performance & Index-Friendly SQL

### 9.1 Sargable vs non-sargable

#### Good (index-friendly):
```sql
WHERE event_date >= '2026-01-01' AND event_date < '2026-02-01'
```

#### Often bad (function on column):
```sql
WHERE YEAR(event_date) = 2026
```

### 9.2 Covering index idea (SQL Server)

- Filter/join columns in index key
- Select columns in INCLUDE

```sql
CREATE INDEX IX_FactVisit_Date_Department
ON dbo.FactVisit (visit_date, department_sk)
INCLUDE (patient_sk, visit_id, amount);
```

### 9.3 Partitioning mental model (warehouse scale)

- Partition big facts by date (month/quarter) when appropriate
- Keep predicates aligned with partition key

## 10) Patterns for BI Semantic Layers

### 10.1 “Metric-ready” view

```sql
CREATE VIEW dbo.vw_VisitMetrics AS
SELECT
  f.visit_id,
  f.visit_date,
  d.department_name,
  p.age_group,
  f.length_of_stay_days,
  f.cost_amount
FROM dbo.FactVisit f
JOIN dbo.DimDepartment d ON f.department_sk = d.department_sk
JOIN dbo.DimPatient    p ON f.patient_sk    = p.patient_sk
WHERE f.is_valid = 1;
```

### 10.2 Pre-aggregate table for Power BI

```sql
SELECT
  CAST(visit_date AS date) AS d,
  department_sk,
  COUNT(*) AS visits,
  SUM(cost_amount) AS total_cost
INTO dbo.AggDailyVisits
FROM dbo.FactVisit
GROUP BY CAST(visit_date AS date), department_sk;
```

## 11) Handy SQL Server Features (optional but useful)

### 11.1 JSON extraction

```sql
SELECT
  JSON_VALUE(payload, '$.user.id') AS user_id,
  JSON_VALUE(payload, '$.event')   AS event_type
FROM dbo.RawEvents;
```

### 11.2 STRING_AGG

```sql
SELECT
  department_sk,
  STRING_AGG(provider_name, ', ') WITHIN GROUP (ORDER BY provider_name) AS providers
FROM dbo.DimProvider
GROUP BY department_sk;
```

## 12) Common Gotchas

- COUNT(col) ignores NULL; COUNT(*) counts rows
- LEFT JOIN + WHERE right_table.col = ... can unintentionally turn into an inner join
- Always define “latest” with deterministic tie-breakers (ORDER BY ts DESC, id DESC)
- Always specify columns in INSERT (never rely on table order)
