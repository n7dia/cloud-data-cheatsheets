# Power BI Modeling Cheatsheet
## (Ordered: Most Important → Least Important)

> Focus: what actually matters to excel as a BI Developer.
> Priority is based on real-world enterprise BI work.

---

## Star Schema (Most Important Concept)

Power BI models should follow a **star schema**.

### Structure

DimDate
|
DimPatient — FactVisits — DimDepartment


### Fact Table
- Contains metrics and events
- High row count
- Numeric measures

Examples:
- Visits
- Costs
- Claims
- Admissions

### Dimension Tables
- Descriptive attributes
- Used for filtering and slicing

Examples:
- Date
- Patient
- Department
- Provider

### Key Rule
> Model first. Visuals become easy later.

---

## Relationships (Correct Modeling)

### Recommended
- One-to-many relationships
- Dimension (1) → Fact (*)
- Single-direction filtering

### Avoid
- Many-to-many relationships
- Bi-directional filters unless required
- Snowflake schemas when possible

### Example

DimDate[DateKey] 1 → * FactVisits[DateKey]


---

## Measures vs Calculated Columns

### Prefer Measures

Measures are dynamic and respect filter context.

```DAX
Total Visits = COUNTROWS(FactVisits)
Total Cost = SUM(FactVisits[Cost])

Avoid Excess Calculated Columns

    Increase model size

    Reduce performance

    Often unnecessary

Rule

    If logic is aggregated → use a measure.

Filter Context & CALCULATE()

Filter context determines measure results.
Example

Visits (Emergency) =
CALCULATE(
    [Total Visits],
    FactVisits[VisitType] = "Emergency"
)

Mental Model

    Slicers change context

    CALCULATE modifies context

Date Table (Required for Time Intelligence)

Create a dedicated date table.

Date =
ADDCOLUMNS(
    CALENDAR(DATE(2020,1,1), DATE(2030,12,31)),
    "Year", YEAR([Date]),
    "Month", FORMAT([Date], "MMM"),
    "YearMonth", FORMAT([Date], "YYYY-MM")
)

Then mark as:

Table Tools → Mark as Date Table

Core Time Intelligence Measures
Year-to-date

Visits YTD =
TOTALYTD([Total Visits], 'Date'[Date])

Previous Year

Visits LY =
CALCULATE(
    [Total Visits],
    SAMEPERIODLASTYEAR('Date'[Date])
)

YoY %

YoY % =
DIVIDE(
    [Total Visits] - [Visits LY],
    [Visits LY],
    0
)

Model Layout & Organization
Best Practices

    Create a dedicated Measures table

    Hide technical keys

    Use business-friendly names

Example Naming

Tables:

FactVisits
DimPatient
DimDate

Measures:

Total Visits
Avg Length of Stay
Readmission Rate %

Power Query vs DAX (Where Logic Lives)
Power Query (ETL layer)

Use for:

    Data cleaning

    Type conversions

    Joins and merges

    Static transformations

DAX (Semantic layer)

Use for:

    Metrics

    Aggregations

    Context-aware calculations

Rule

    Transform rows in Power Query, calculate metrics in DAX.

Performance Basics
Model Size Reduction

    Remove unused columns

    Prefer numeric keys

    Reduce high-cardinality columns

    Disable auto date/time

Cardinality Awareness

High cardinality (expensive):

    IDs

    Free text

    Long strings

Lower cardinality (better):

    Categories

    Codes

    Groups

Import vs DirectQuery
Mode	Recommended Use
Import	Default, fastest
DirectQuery	Very large datasets
Composite	Mixed scenarios

Rule:

    Import mode is usually best for analytics.

Row-Level Security (RLS)

Used for controlled access.

Example:

[DepartmentID] = USERPRINCIPALNAME()

Common scenarios:

    Department-based access

    Team-level dashboards

Aggregation Tables (Advanced Optimization)

Create summarized tables for large datasets.

Example:

FactVisits (granular)
AggDailyVisits (summarized)

Used to improve performance on large models.
Common Modeling Mistakes

    Flat single-table models

    Too many calculated columns

    Bi-directional filters everywhere

    Building visuals before modeling

    Exposing technical keys to users

BI Developer Workflow (Reality)

1. Understand business metric
2. Define grain
3. Build star schema
4. Create relationships
5. Build measures
6. Optimize model
7. Create visuals

Quick Checklist Before Publishing

✔ Star schema used
✔ Date table created
✔ Measures instead of columns
✔ Relationships clean
✔ Keys hidden
✔ Model optimized
✔ Naming consistent

Core Principle
Good model = simple visuals.
Bad model = complex visuals.