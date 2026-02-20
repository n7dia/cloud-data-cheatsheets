# KQL CheatSheet (Kusto Query Language)

## Basic Shape
```
TableName
| where ...
| summarize ...
| order by ...
```

## 1) Filter + Select Columns
```
Events
| where Timestamp >= ago(7d)
| where EventLevel == "Error"
| project Timestamp, UserId, Message
```

## 2) String Search
```
Events
| where Message has "timeout"          // token-based
| where Message contains "time"        // substring
| where Message matches regex @"t(ime|o)ut"
```

## 3) Parse / Extract
```
Events
| parse Message with * "user=" userId " " * "ip=" ip " " *
| project Timestamp, userId, ip
```

## 4) Create Columns
```
Events
| extend Day = startofday(Timestamp),
         IsProd = Environment == "prod"
```

## 5) Aggregations
```
Events
| where Timestamp >= ago(30d)
| summarize Errors=count() by bin(Timestamp, 1d)
| order by Timestamp asc
```

## 6) Distinct / Top
```
Events
| summarize Users=dcount(UserId)
```
```
Events
| summarize Cnt=count() by ErrorCode
| top 10 by Cnt desc
```

## 7) Joins
```
Events
| where Timestamp >= ago(7d)
| join kind=leftouter (
    Users | project UserId, Country
  ) on UserId
| summarize Cnt=count() by Country
```

## 8) Time Series (bin + make-series)
```
Events
| where Timestamp >= ago(14d)
| make-series Cnt=count() default=0 on Timestamp step 1d
```

## 9) Nulls + Type Conversions
```
Events
| extend Lat = todouble(LocationLat)
| where isnotnull(Lat)
```
```
Events
| extend Id = toint(UserId)
```

## 10) Helpful Operators

- project (select), extend (create), where (filter)
- summarize (group/agg), join, union
- mv-expand (explode arrays), render (visualize in tools that support it)
