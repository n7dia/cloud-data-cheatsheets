# Spark (PySpark) Cheatsheet


## Setup + Session
```
from pyspark.sql import SparkSession, functions as F, types as T
```
```
spark = (SparkSession.builder
         .appName("cheatsheet")
         .getOrCreate())
```

## 1) Read / Write
```
df = spark.read.parquet("path")
df = spark.read.json("path")
df = spark.read.csv("path", header=True, inferSchema=True)
```

```
df.write.mode("overwrite").parquet("out_path")
df.write.mode("append").format("delta").save("out_delta")   # if Delta enabled
```

## 2) Quick Inspect
```
df.printSchema()
df.show(20, truncate=False)
df.describe().show()
df.count()
df.select("col").distinct().count()
```

## 3) Select, Rename, Cast, Create Columns
```
df2 = (df
  .select("a", "b", F.col("c").alias("c_new"))
  .withColumn("d_int", F.col("d").cast("int"))
  .withColumn("flag", F.when(F.col("x") > 0, 1).otherwise(0))
)
```

## 4) Filter + Null Handling
```
df.filter(F.col("status") == "active")
df.filter(F.col("x").isNull())
df.na.fill({"city": "Unknown"})
df.dropna(subset=["id"])
```

## 5) GroupBy + Aggregations
```
agg = (df.groupBy("dept")
       .agg(F.count("*").alias("n"),
            F.avg("salary").alias("avg_salary"),
            F.sum("salary").alias("total_salary")))
```

## 6) Joins (very common)
```
joined = (fact.alias("f")
  .join(dim.alias("d"), on=F.col("f.dim_id")==F.col("d.id"), how="left")
  .select("f.*", "d.dim_name"))
```

## 7) Window Functions
```
from pyspark.sql.window import Window
```
```
w = Window.partitionBy("customer_id").orderBy(F.col("order_date").desc())
df_ranked = df.withColumn("rn", F.row_number().over(w)).filter("rn = 1")
```

## 8) Deduplicate
```
df.dropDuplicates(["id"])
```
- Or explicitly keep the newest row using window functions

## 9) Dates + Times
```
df = (df
  .withColumn("dt", F.to_date("date_str"))
  .withColumn("month", F.date_format("dt", "yyyy-MM"))
)
```

## 10) Performance Tips

- Prefer built-in functions (F.*) over Python UDFs.
- Watch join type + data skew; consider broadcast for small dimensions:

      ```
      from pyspark.sql.functions import broadcast
      fact.join(broadcast(dim), "key", "left")
      ```

- Cache only when reused:
      ```
      df.cache()
      ```
