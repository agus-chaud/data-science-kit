# Structured Streaming cookbook

Loaded from `SKILL.md` when the task touches streaming: Kafka sources, Auto Loader, triggers, watermarks, `foreachBatch` upserts.

Ground rules: one checkpoint per query, define event-time + watermark explicitly, choose trigger mode by workload shape (continuous cadence vs backfill catch-up).

## Kafka source with explicit schema and watermark

```python
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, TimestampType
from delta.tables import DeltaTable

events_schema = StructType([
    StructField("event_id", StringType()),
    StructField("customer_id", StringType()),
    StructField("event_type", StringType()),
    StructField("event_time", TimestampType()),
    StructField("amount", DoubleType()),
])

# Kafka source with explicit offsets (use earliest for controlled reprocessing/backfills)
raw_kafka = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "broker1:9092,broker2:9092")
    .option("subscribe", "orders-events")
    .option("startingOffsets", "latest")
    .load()
)

parsed = (
    raw_kafka
    .select(F.from_json(F.col("value").cast("string"), events_schema).alias("data"))
    .select("data.*")
    .withWatermark("event_time", "10 minutes")
)
```

## Auto Loader (`cloudFiles`) with schema tracking

```python
auto_loader_df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "dbfs:/checkpoints/bronze/orders_schema")
    .load("s3://company-landing/orders/")
)
```

## Triggers

Use `processingTime` for near-real-time cadence; use `availableNow=True` in Jobs for bounded incremental catch-up.

```python
# Continuous micro-batch cadence
agg_query = (
    parsed.groupBy(F.window("event_time", "5 minutes"), "event_type")
    .agg(F.count("*").alias("event_count"), F.sum("amount").alias("total_amount"))
    .writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "dbfs:/checkpoints/silver/event_metrics")
    .trigger(processingTime="1 minute")
    .table("prod.silver.event_metrics")
)

# Job-friendly bounded run (process all available data and stop)
available_now_query = (
    parsed.writeStream
    .format("delta")
    .option("checkpointLocation", "dbfs:/checkpoints/bronze/orders_events")
    .trigger(availableNow=True)
    .table("prod.bronze.orders_events")
)
```

## `foreachBatch` + Delta MERGE (upserts into Silver/Gold)

Native `writeStream` sinks are limited for complex upserts — inside each batch, treat the micro-batch as a static DataFrame and run `MERGE`/multi-table writes.

```python
def upsert_orders(micro_batch_df, batch_id):
    if micro_batch_df.isEmpty():
        return

    # Avoid recomputation when the same micro-batch is read multiple times.
    cached = micro_batch_df.cache()

    delta_target = DeltaTable.forName(spark, "prod.silver.orders")
    (
        delta_target.alias("t")
        .merge(cached.alias("s"), "t.event_id = s.event_id")
        .whenMatchedUpdate(
            condition="s.event_time >= t.event_time",
            set={
                "customer_id": "s.customer_id",
                "event_type": "s.event_type",
                "amount": "s.amount",
                "event_time": "s.event_time",
            },
        )
        .whenNotMatchedInsertAll()
        .execute()
    )

    cached.unpersist()

upsert_query = (
    parsed.writeStream
    .foreachBatch(upsert_orders)
    .option("checkpointLocation", "dbfs:/checkpoints/silver/orders_merge")
    .trigger(processingTime="2 minutes")
    .start()
)
```

## Monitoring

Monitor with `query.lastProgress`, `query.recentProgress`, and streaming metrics in Spark UI / Databricks Jobs run output.
