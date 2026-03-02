# Dimensional Modelling Cheatsheet

Practical dimensional modeling for Data Warehousing, BI, and Analytics.

Goal: build clean, analytics-ready models that scale.


## Core Idea

Dimensional modeling organizes data for analytics using:

Fact Tables + Dimension Tables


Designed for:
- Fast querying
- Simple BI analysis
- Clear business logic


## Star Schema (Primary Pattern)

### Structure

        DimDate
            |
DimPatient — FactVisits — DimDepartment
            |
        DimProvider


### Principles
- Fact table in the center
- Dimensions surround facts
- Simple joins
- Optimized for analytics



## Fact Tables

Contain measurable events.

### Characteristics
- High row count
- Numeric metrics
- Foreign keys to dimensions
- Defined grain

### Examples
- FactVisits
- FactAdmissions
- FactOrders
- FactClaims

### Example Structure

```sql
FactVisits
------------
visit_sk
date_sk
patient_sk
department_sk
cost_amount
length_of_stay
```

Grain (Most Important Concept)

Grain = what one row represents.

Examples:

    One visit

    One transaction

    One patient per day

Rule

    Define grain BEFORE adding columns.

Bad modeling starts when grain is unclear.
Dimension Tables

Provide descriptive context.
Characteristics

    Lower row count

    Text attributes

    Used for filtering and grouping

Examples

    DimDate

    DimPatient

    DimDepartment

    DimLocation

Example

DimDepartment
---------------
department_sk
department_name
facility
service_type

Keys
Surrogate Key (Recommended)

System-generated integer key.

patient_sk INT IDENTITY

Why:

    Stable

    Efficient joins

    Handles changing source IDs

Business Key

Natural identifier from source system.

Example:

patient_id
employee_id
order_number

Store in dimension but do not use as join key.
Fact vs Dimension Summary
Feature	Fact Table	Dimension Table
Purpose	Metrics	Context
Row count	High	Low
Numeric values	Yes	Rarely
Used for filtering	No	Yes
Keys	Foreign keys	Primary surrogate key
Slowly Changing Dimensions (SCD)

Used when descriptive data changes over time.
Type 1 (Overwrite)

    Replace old value

    No history kept

Example:

Address correction

Type 2 (History Tracking)

    Create new row when attributes change

    Preserve history

Additional columns:

valid_from
valid_to
is_current

Example:

Patient changes postal code

Type 3 (Limited History)

    Store current + previous value

    Used rarely

Fact Table Types
Transaction Fact

One row per event.

Examples:

    Visits

    Orders

    Purchases

Snapshot Fact

State of data at intervals.

Examples:

    Daily occupancy

    Monthly balances

Accumulating Snapshot

Tracks lifecycle stages.

Example:

Admission → Treatment → Discharge

Degenerate Dimensions

Identifiers stored directly in fact table.

Example:

invoice_number
visit_id

Used when no additional attributes exist.
Role-Playing Dimensions

Same dimension used multiple times.

Example:

DimDate
  - AdmissionDate
  - DischargeDate
  - BillingDate

Bridge Tables (Advanced)

Used for many-to-many relationships.

Example:

Patient ↔ Multiple Diagnoses

Structure:

Fact ↔ Bridge ↔ Dimension

Use only when necessary.
Slowly Changing Fact Considerations

Facts usually:

    are immutable

    represent historical truth

Avoid updating facts unless correcting errors.
Modeling Workflow (Real-World)

1. Understand business process
2. Define grain
3. Identify metrics (facts)
4. Identify dimensions
5. Create surrogate keys
6. Load dimensions first
7. Load fact table

Common Modeling Mistakes

    Undefined grain

    Mixing multiple grains in one table

    Over-normalized schemas

    Too many joins

    Using business keys as primary joins

    Putting descriptive text in fact tables

Performance Best Practices

    Use integer surrogate keys

    Keep fact tables narrow

    Avoid unnecessary columns

    Partition large facts by date

    Flatten dimensions when possible

Quick Checklist

✔ Grain clearly defined
✔ Star schema design
✔ Facts contain metrics
✔ Dimensions contain attributes
✔ Surrogate keys used
✔ History strategy defined
Core Principle

Facts measure.
Dimensions describe.

BI & Data Engineering Insight

Good dimensional models make:

    SQL simpler

    Power BI faster

    Metrics consistent

    Pipelines easier to maintain