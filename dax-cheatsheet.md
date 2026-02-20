# DAX Cheatsheet (Power BI)

Core Ideas: 
- Calculated column: computed row-by-row at refresh.
- Measure: computed at query-time, respects filter context.
- Filter context changes results; CALCULATE() edits filter context.


## 1) Basic Measures
```
Total Sales = SUM(Sales[SalesAmount])
Total Orders = COUNTROWS(Sales)
Avg Order Value = DIVIDE([Total Sales], [Total Orders])
```

## 2) DISTINCTCOUNT, Safe Division
```
Customers = DISTINCTCOUNT(Sales[CustomerID])
Margin % = DIVIDE([Total Profit], [Total Sales], 0)
```

## 3) CALCULATE + Filters
```
Sales (Online Only) =
CALCULATE([Total Sales], Sales[Channel] = "Online")
```

## 4) Remove / Keep Filters
```
Sales (All Regions) =
CALCULATE([Total Sales], ALL(Region))
```

```
Sales (Keep Product Filter, Ignore Date) =
CALCULATE([Total Sales], ALLEXCEPT(Sales, Sales[ProductID]))
```

## 5) Filter a Table (common pattern)
```
High Value Sales =
CALCULATE(
  [Total Sales],
  FILTER(Sales, Sales[SalesAmount] > 1000)
)
```

## 6) Time Intelligence (requires a Date table)
```
Sales YTD = TOTALYTD([Total Sales], 'Date'[Date])
Sales MTD = TOTALMTD([Total Sales], 'Date'[Date])
Sales YoY =
CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
```
```
YoY % =
DIVIDE([Total Sales] - [Sales YoY], [Sales YoY], 0)
```

## 7) Rolling / Moving Window
```
Sales Last 30 Days =
CALCULATE(
  [Total Sales],
  DATESINPERIOD('Date'[Date], MAX('Date'[Date]), -30, DAY)
)
```

## 8) Rank (Top N)
```
Product Rank =
RANKX(ALL(Product[ProductName]), [Total Sales], , DESC)
```

## 9) Common “Share of Total”
```
Sales % of Total =
DIVIDE(
  [Total Sales],
  CALCULATE([Total Sales], ALL(Product)),
  0
)
```

## 10) Debugging Tips

- If a measure is “wrong”, check: filter context + relationships + granularity.
- Prefer DIVIDE() over /.
- Always create a proper Date table and mark it as Date table.
