# Delta Lake cookbook

Loaded from `SKILL.md` when the task touches Delta tables: creation, MERGE/upserts, time travel, optimization, CDF, clones.

## Creating and managing Delta tables

```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable
from pyspark.sql.functions import col, current_timestamp, expr

spark = SparkSession.builder.getOrCreate()

# Create Delta table
df = spark.read.json("/mnt/raw/events")
df.write.format("delta") \
    .mode("overwrite") \
    .option("overwriteSchema", "true") \
    .partitionBy("date", "event_type") \
    .save("/mnt/delta/events")

# Create managed table
df.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("production.events")

# Create table with explicit LOCATION (external-location pattern)
spark.sql("""
    CREATE TABLE IF NOT EXISTS production.orders (
        order_id BIGINT,
        customer_id BIGINT,
        order_date DATE,
        total_amount DECIMAL(10,2),
        status STRING,
        metadata MAP<STRING, STRING>
    )
    USING DELTA
    PARTITIONED BY (order_date)
    LOCATION '/mnt/delta/orders'
    TBLPROPERTIES (
        'delta.autoOptimize.optimizeWrite' = 'true',
        'delta.autoOptimize.autoCompact' = 'true'
    )
""")

# Add constraints
spark.sql("""
    ALTER TABLE production.orders
    ADD CONSTRAINT valid_status CHECK (status IN ('pending', 'completed', 'cancelled'))
""")

# Add generated columns
spark.sql("""
    ALTER TABLE production.orders
    ADD COLUMN month INT GENERATED ALWAYS AS (MONTH(order_date))
""")
```

## MERGE operations (upserts)

```python
from delta.tables import DeltaTable

delta_table = DeltaTable.forPath(spark, "/mnt/delta/orders")
updates_df = spark.read.format("parquet").load("/mnt/staging/order_updates")

# Merge (upsert)
delta_table.alias("target").merge(
    updates_df.alias("source"),
    "target.order_id = source.order_id"
).whenMatchedUpdate(
    condition="source.updated_at > target.updated_at",
    set={
        "total_amount": "source.total_amount",
        "status": "source.status",
        "updated_at": "source.updated_at"
    }
).whenNotMatchedInsert(
    values={
        "order_id": "source.order_id",
        "customer_id": "source.customer_id",
        "order_date": "source.order_date",
        "total_amount": "source.total_amount",
        "status": "source.status",
        "created_at": "source.created_at",
        "updated_at": "source.updated_at"
    }
).execute()

# Merge with delete
delta_table.alias("target").merge(
    updates_df.alias("source"),
    "target.order_id = source.order_id"
).whenMatchedUpdate(
    condition="source.is_active = true",
    set={"status": "source.status"}
).whenMatchedDelete(
    condition="source.is_active = false"
).whenNotMatchedInsert(
    values={
        "order_id": "source.order_id",
        "status": "source.status"
    }
).execute()
```

For streaming upserts (`foreachBatch` + MERGE), see `streaming.md`.

## Time travel and versioning

```python
# Query historical versions
df_v0 = spark.read.format("delta").option("versionAsOf", 0).load("/mnt/delta/orders")
df_yesterday = spark.read.format("delta") \
    .option("timestampAsOf", "2024-01-15") \
    .load("/mnt/delta/orders")

# View history
delta_table = DeltaTable.forPath(spark, "/mnt/delta/orders")
delta_table.history().show()

# Restore to previous version
delta_table.restoreToVersion(5)
delta_table.restoreToTimestamp("2024-01-15")

# Vacuum old files (delete files older than retention period)
delta_table.vacuum(168)  # 7 days in hours — respect retention before vacuuming

# View table details
delta_table.detail().show()
```

## Optimization and maintenance

```python
# Optimize table (compaction)
spark.sql("OPTIMIZE production.orders")

# Optimize with Z-Ordering — pick columns that appear in filter predicates
spark.sql("OPTIMIZE production.orders ZORDER BY (customer_id, status)")

# Analyze table for statistics
spark.sql("ANALYZE TABLE production.orders COMPUTE STATISTICS")

# Clone table (zero-copy)
spark.sql("""
    CREATE TABLE production.orders_clone
    SHALLOW CLONE production.orders
""")

# Deep clone (independent copy)
spark.sql("""
    CREATE TABLE production.orders_backup
    DEEP CLONE production.orders
""")
```

**Anti-pattern — many small files:** appending file-by-file creates a small-file explosion that OPTIMIZE then has to clean up.

```python
# Bad: one append per input file
for file in files:
    df = spark.read.json(file)
    df.write.format("delta").mode("append").save("/mnt/table")

# Good: batch writes with optimized writes enabled
df = spark.read.json("/mnt/source/*")
df.write.format("delta") \
    .option("optimizeWrite", "true") \
    .mode("append") \
    .save("/mnt/table")
```

## Change Data Feed (CDC / incremental processing)

```python
spark.sql("""
    ALTER TABLE production.orders
    SET TBLPROPERTIES (delta.enableChangeDataFeed = true)
""")

# Read changes
changes_df = spark.read.format("delta") \
    .option("readChangeFeed", "true") \
    .option("startingVersion", 5) \
    .table("production.orders")
```
