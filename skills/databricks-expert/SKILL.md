---
name: databricks-expert
version: 1.1.0
description: Databricks reference. Use when the user works with PySpark, Delta Lake, Unity Catalog, cluster config or DBU cost, Structured Streaming, or MLflow on Databricks — or mentions medallion, lakehouse, DBR, Auto Loader, foreachBatch, Z-order.
category: data
author: PCL Team
license: Apache-2.0
tags:
  - databricks
  - spark
  - delta-lake
  - mlflow
  - lakehouse
  - pyspark
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Glob
  - Grep
requirements:
  databricks-sdk: ">=0.20.0"
  pyspark: ">=3.4.0"
---

# Databricks Expert

Core concepts and decision rules live here. Code cookbooks live in sibling files — read the one whose branch the task touches:

| Task touches | Read |
|---|---|
| Delta tables: creation, MERGE/upserts, time travel, OPTIMIZE/Z-order, CDF, clones | `delta.md` |
| Structured Streaming: Kafka, Auto Loader, triggers, watermarks, `foreachBatch` upserts | `streaming.md` |
| Unity Catalog: grants, external locations, lineage, federation, Delta Sharing, audit, masking | `unity-catalog.md` |
| MLflow: tracking, registry (UC + aliases), batch inference | `mlflow.md` |

## Lakehouse and Medallion architecture

The **lakehouse** combines cheap open storage (data lake) with warehouse-style reliability (transactions, governance). **Medallion** is the usual layout: three quality tiers so data improves as it flows through the system.

| Layer | Role | Typical contents |
|-------|------|-------------------|
| **Bronze** | System of ingestion | Raw or minimally typed data, append-friendly history, source-faithful payloads (JSON/CSV blobs, CDC mirrors). Prefer not to destroy history here. |
| **Silver** | Conformed, clean layer | Standardized types (e.g. strings → dates), deduplicated keys, business rules, shared dimensions. Tables here should be safe for analysts to query. |
| **Gold** | Consumption layer | Aggregates, KPIs, wide feature sets, curated datasets for BI, ML training/serving, and executive reporting. |

**Implementation notes:** Bronze often uses idempotent batch loads (e.g. Delta **COPY INTO**) or streaming ingest (Auto Loader, Kafka). Silver/Gold frequently combine **Structured Streaming** or micro-batch jobs with **Delta MERGE** (upserts) via the **`foreachBatch`** pattern — see `streaming.md`. Pair with Unity Catalog three-level names (`catalog.schema.table`).

## Workspace, notebooks, and file formats

**Notebook shortcuts (Databricks):**

- **Shift + Enter** — run the current cell and move to the next.
- **Ctrl + Enter** (macOS: **Cmd + Enter**) — run the current cell without moving the cursor.

**Inspecting results:**

- **`display(df)`** (Databricks) — interactive table, charts, and export-friendly exploration in the notebook UI.
- **`df.show()` / `df.show(n)`** — plain-text preview in the cell output; fine for logs and quick checks, not for rich exploration.

**Formats for pipelines:** Prefer **Delta Lake** (default on Databricks; ACID, time travel, MERGE) or **Parquet** for analytical storage. Use **CSV/JSON** mainly at boundaries (exports, legacy feeds, web APIs); they are flexible but expensive to scan at scale. For Medallion flows, land raw in Bronze (any format), converge to **Delta** by Silver at the latest so downstream jobs get predictable performance and governance.

## Platform architecture: control plane and data plane

Databricks splits **management** (control plane) from **processing and storage** (data plane) on typical **enterprise** deployments:

- **Control plane** — Databricks-hosted UI, APIs, job scheduling, cluster manager, Unity Catalog metadata, etc. Your notebook code and metadata live here; **your table bytes usually do not**.
- **Data plane** — Customer-owned cloud storage (S3, ADLS, GCS) and compute (VMs) running in **your** cloud account or delegated subscription. Clusters read/write datasets here; data stays under **your** security and billing boundary.

**Why this split matters:**

1. **Security** — Sensitive payloads stay in storage you control; access is via credentials and UC policies instead of shipping data through Databricks servers.
2. **Compliance** — Easier to meet GDPR, HIPAA, SOX when residency and encryption keys align with your cloud contracts.
3. **Control** — You keep IAM/RBAC, networking (VPC, private endpoints), and backup strategy aligned with enterprise standards.
4. **Cost transparency** — You pay the cloud provider for storage and often for compute VMs; Databricks bills **DBUs** for the platform layer (exact model varies by contract and product).

**Typical flow:** User → control plane (start job, open notebook) → cluster starts in data plane → cluster reads/writes **your** object storage → results written back to **your** storage.

**Free Edition / simplified trials:** Databricks may operate a **fully managed** stack (control + data) so you can learn without wiring a cloud account. Treat that as a **learning default**, not the same topology as production enterprise lakehouses.

```mermaid
flowchart LR
  userNode[User] --> cp[ControlPlane]
  cp --> clusterNode[SparkCluster]
  clusterNode --> storageNode[CustomerCloudStorage]
```

## Compute model (Databricks Runtime and serverless)

**Mental model:** *compute* is where Spark executes; **Databricks Runtime (DBR)** is the curated stack (Apache Spark + connectors + Databricks optimizations + patched dependencies) identified by strings such as `13.3.x-scala2.12`.

### Serverless (high level)

**Serverless** offerings (for example **serverless SQL warehouses** and **serverless jobs / notebooks** where enabled) are **fully managed**: Databricks provisions and scales capacity; you configure warehouse or job settings instead of owning a long-lived classic cluster object. Exact features, regions, SLAs, and pricing depend on **entitlements and roadmap**—always confirm against current product docs for your workspace rather than assuming parity with classic compute.

### Runtime families

- **Standard** — Default DBR for general Spark SQL / DataFrame / batch ETL.
- **ML** — Adds curated ML/DL libraries and integrations; choose when you depend on that bundle instead of hand-vendoring identical wheels on Standard.
- **Photon** — Vectorized execution engine (C++); can reduce wall-clock for eligible SQL/DataFrame workloads but is billed at a **different DBU multiplier** than non-Photon—benchmark before fleet-wide enablement.
- **GPU** — Hardware for deep learning / CUDA; pair with ML-oriented images when training/serving needs GPUs.

### LTS vs Latest

- **LTS (Long-Term Support)** — Slower cadence of breaking changes across the support window; preferred **default for production jobs** when upgrade testing is costly.
- **Latest** (non-LTS) — Picks up newer Spark/features sooner; great for **sandboxes** and early validation, weaker as an unvetted default for immovable SLAs.

### DBR change / compatibility warning

Bumping `spark_version` is a **release event**, not a config tweak. Expect possible breaks from: Spark behavior changes, connector/JAR ABI shifts, Python/Scala standard library bumps, removed Databricks flags, and notebook dependencies pinned to older images. **Pin** DBR per environment (dev/stage/prod), read Databricks **release notes** for the jump, run integration tests, and roll out gradually (canary job cluster + monitoring) instead of upgrading every cluster at once.

## Cluster roles, economics, policies, and pools

### Driver vs workers

- **Driver** — Hosts `SparkContext`, scheduler, and session state; executes `collect()`, `toPandas()`, and driver-side Python orchestration. Driver OOM or failure **terminates the whole application** even if executors are healthy.
- **Workers (executors)** — Run parallel tasks; scale out for CPU/shuffle-heavy stages until skew, shuffle partitions, or the driver becomes the bottleneck.

Use a **larger `driver_node_type_id` than `node_type_id`** only when profiling shows driver pressure (huge broadcasts, massive task metadata, heavy driver Python). Oversized drivers burn DBUs without helping executors.

### Job clusters vs all-purpose clusters

| Mode | Lifecycle | Cost angle |
|------|-----------|------------|
| **Job / workflow cluster** | Created for a **run** (or task attempt) and torn down when idle policy says so. | You mostly pay for **active pipeline minutes**—the default for scheduled ETL, ML training, and production workflows. |
| **All-purpose (shared) cluster** | Long-lived for notebooks, BI, ad-hoc SQL; optional autotermination. | **Idle time still bills** if left up—convenient for humans, expensive as a default for unattended workloads. |

**Heuristic:** automated pipelines → **job** compute; human exploration → **all-purpose** with **aggressive autotermination** or **serverless SQL** where appropriate. For fault-tolerant workloads, **Spot/Preemptible instances** cut VM cost further.

### Autoscale and autotermination

- **Autoscale** (`min_workers` / `max_workers`) — Adds executors under load; keep a **sane max** to cap spend and watch for stragglers / skew that autoscale cannot fix alone.
- **Autotermination** — Stops interactive clusters after *N* minutes idle; typical non-prod shared clusters use **15–30 minutes** (tighten further if analysts forget to terminate).

### DBUs — what drives the bill

**DBUs (Databricks Units)** meter platform consumption; effective **DBU/hour** depends on **SKU**, **region**, **Photon**, **ML/GPU** runtime multipliers, and **contract discounts**. Cloud VM/storage charges may appear on your **cloud invoice** separately depending on deployment model—FinOps should reconcile **both** Databricks usage exports and cloud cost tags. Instrument runs with **tags** so attribution is possible.

### Tags, cluster policies, and instance pools (single story)

- **Tags** (`custom_tags`, job/workflow tags) — Carry **team**, **cost_center**, **environment**, **data_domain** into usage reports for chargeback and anomaly detection.
- **Cluster policies** — Admin-defined JSON guardrails: **fixed** values, **allow-lists** (approved `node_type_id` / DBR versions), **deny patterns**, and **numeric ranges** (worker min/max). Policies prevent "surprise GPU" clusters while still delegating safe self-service.
- **Instance pools** — Hold **pre-warmed** capacity on blessed SKUs (`min_idle_instances`, `max_capacity`, `preloaded_spark_versions`) so clusters attach quickly; idle pool capacity has **its own cost trade-off** versus cold start latency.

**Together:** policies define **what** users may request; pools supply **where** approved capacity comes from; tags explain **who** must pay.

### Cluster configuration snippets

```python
# Databricks CLI - Create cluster
databricks clusters create --json '{
  "cluster_name": "data-engineering-cluster",
  "spark_version": "13.3.x-scala2.12",
  "node_type_id": "i3.xlarge",
  "autoscale": {
    "min_workers": 2,
    "max_workers": 8
  },
  "autotermination_minutes": 120,
  "spark_conf": {
    "spark.sql.adaptive.enabled": "true",
    "spark.databricks.delta.optimizeWrite.enabled": "true"
  },
  "custom_tags": {
    "team": "data-engineering",
    "environment": "production"
  }
}'

# Instance pool + cluster attached to it
instance_pool_config = {
    "instance_pool_name": "production-pool",
    "min_idle_instances": 2,
    "max_capacity": 20,
    "node_type_id": "i3.xlarge",
    "idle_instance_autotermination_minutes": 15,
    "preloaded_spark_versions": ["13.3.x-scala2.12"]
}

cluster_with_pool = {
    "cluster_name": "pool-cluster",
    "spark_version": "13.3.x-scala2.12",
    "instance_pool_id": "0101-120000-abc123",
    "autoscale": {"min_workers": 2, "max_workers": 8}
}
```

## Apache Spark fundamentals

Spark is commonly described as five cooperating modules (historical naming still useful for mental models):

1. **Spark Core** — distributed execution engine (RDD lineage, scheduler, fault tolerance).
2. **Spark SQL** — structured APIs (`DataFrame`, `Dataset`) + Catalyst optimizer + SQL parser.
3. **Structured Streaming** — incremental engine built on Spark SQL logical plans.
4. **MLlib** — scalable ML pipelines and feature transformers.
5. **GraphX / GraphFrames ecosystem** — graph processing use cases (less common in modern lakehouse ETL, but still relevant in some domains).

**SparkSession API surface (what you touch daily):**

- `spark.read` / `spark.readStream` — ingest batch or streaming sources.
- `spark.sql(...)` — SQL execution over managed/external tables.
- `spark.table("catalog.schema.table")` — direct DataFrame from UC table.
- `spark.catalog` — metadata discovery utilities.
- `spark.conf` — runtime/session-level Spark settings.

### Lazy evaluation: transformations vs actions

Spark builds a logical plan lazily and executes only when an **action** is triggered.

| Type | Examples | Notes |
|------|----------|-------|
| **Transformations** | `select`, `withColumn`, `filter`, `join`, `groupBy`, `orderBy`, `dropDuplicates` | Return a new DataFrame lineage; no cluster job yet. |
| **Actions** | `show`, `count`, `collect`, `take`, `write`, `saveAsTable`, `foreachBatch` | Trigger execution and materialize results/side effects. |

**Practical implication:** chain transformations first, then minimize actions — especially `collect()`, which pulls the full dataset onto the driver:

```python
# Bad: collect large dataset to driver → driver OOM
large_df.collect()

# Good: use actions that stay distributed
large_df.write.format("delta").save("/mnt/output")
```

## DataFrame and SQL patterns

### Join patterns (including semi/anti)

```python
from pyspark.sql import functions as F

orders = spark.table("prod.sales.orders")
customers = spark.table("prod.sales.dim_customers")
blocked = spark.table("prod.sales.blocked_customers")

# INNER / LEFT / RIGHT / FULL use the same `join(..., how=...)` shape.
enriched = orders.join(customers, "customer_id", "left")

# LEFT SEMI: keep rows from left that have at least one match in right (returns left columns only).
with_customer = orders.join(customers.select("customer_id"), "customer_id", "left_semi")

# LEFT ANTI: keep rows from left with no match in right.
not_blocked = orders.join(blocked.select("customer_id"), "customer_id", "left_anti")
```

### Aggregations: groupBy / agg / pivot / rollup / cube

```python
sales = spark.table("prod.sales.orders")

summary = (
    sales.groupBy("country", "order_status")
    .agg(
        F.count("*").alias("orders"),
        F.sum("total_amount").alias("revenue"),
        F.avg("total_amount").alias("avg_ticket"),
    )
)

pivoted = (
    sales.groupBy("country")
    .pivot("order_status", ["pending", "completed", "cancelled"])
    .agg(F.count("*"))
)

rollup_totals = sales.rollup("country", "order_status").agg(F.sum("total_amount").alias("revenue"))
cube_totals = sales.cube("country", "order_status").agg(F.sum("total_amount").alias("revenue"))
```

### Windows: running total in one line

```python
from pyspark.sql.window import Window

running = spark.table("prod.sales.orders").withColumn(
    "running_total",
    F.sum("total_amount").over(
        Window.partitionBy("customer_id").orderBy("order_date").rowsBetween(Window.unboundedPreceding, Window.currentRow)
    ),
)
```

Equivalent SQL pattern:

```sql
SELECT
  customer_id,
  order_date,
  total_amount,
  SUM(total_amount) OVER (
    PARTITION BY customer_id
    ORDER BY order_date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM prod.sales.orders;
```

### CTEs and segmentation in Spark SQL

```python
df.createOrReplaceTempView("orders_temp")

result = spark.sql("""
    WITH customer_metrics AS (
        SELECT
            customer_id,
            COUNT(*) AS order_count,
            SUM(total_amount) AS lifetime_value,
            DATEDIFF(MAX(order_date), MIN(order_date)) AS customer_age_days,
            COLLECT_LIST(
                STRUCT(order_id, order_date, total_amount)
            ) AS order_history
        FROM orders_temp
        GROUP BY customer_id
    ),
    customer_segments AS (
        SELECT
            *,
            CASE
                WHEN lifetime_value >= 10000 THEN 'VIP'
                WHEN lifetime_value >= 5000 THEN 'Gold'
                WHEN lifetime_value >= 1000 THEN 'Silver'
                ELSE 'Bronze'
            END AS segment,
            NTILE(10) OVER (ORDER BY lifetime_value DESC) AS decile
        FROM customer_metrics
    )
    SELECT * FROM customer_segments
    WHERE segment IN ('VIP', 'Gold')
""")
```

### JSON, arrays, and structs

```python
json_df = df.withColumn("parsed_metadata", F.from_json("metadata", schema))
json_df = json_df.withColumn("tags", F.explode("parsed_metadata.tags"))

df.withColumn("first_item", F.col("items").getItem(0)) \
  .withColumn("item_count", F.size("items")) \
  .withColumn("total_price",
      F.aggregate("items", F.lit(0),
                  lambda acc, x: acc + x.price))
```

## I/O and formats

For lakehouse pipelines, formats are a design choice with long-term cost/performance impact.

| Format | Schema handling | Compression / scan efficiency | ACID / updates | Typical use |
|--------|------------------|-------------------------------|----------------|-------------|
| **CSV** | Weak (string-heavy, manual typing) | Poor to medium | No | Interchange with legacy systems, ad-hoc exports. |
| **JSON** | Flexible / semi-structured | Poor to medium (verbose) | No | API payload capture, raw bronze landing. |
| **Parquet** | Strong typed columnar | High (column pruning + compression) | No native transactions | Analytical batch datasets, non-transactional marts. |
| **Delta** | Parquet + transaction log | High | **Yes** (ACID, schema evolution, time travel, MERGE) | Default for curated Silver/Gold and production ETL. |

**Guideline:** ingest whatever arrives (CSV/JSON) in Bronze if needed, then converge to **Delta** early (Silver latest) to unlock reliable upserts, governance, and performance tooling.

## Performance and reliability

### UDFs vs vectorized operations (and Pandas UDF)

- Prefer built-in Spark SQL functions (`F.when`, `F.expr`, `F.transform`, `F.aggregate`, etc.) over generic Python UDFs.
- Use **Pandas UDF** (`pandas_udf`) only when native functions cannot express the logic and vectorized Arrow batches are acceptable.
- Avoid row-wise Python UDF on large datasets: it breaks Catalyst optimization visibility and typically reduces throughput.

### Broadcast joins and plan inspection

```python
from pyspark.sql import functions as F

fact = spark.table("prod.sales.orders")
dim = spark.table("prod.sales.dim_customers")

# Broadcast the small, stable side — avoids shuffling the large fact table
joined = fact.join(F.broadcast(dim), "customer_id", "left")
joined.explain("formatted")  # verify BroadcastHashJoin / plan shape
```

Use `broadcast(...)` when the right side is genuinely small and stable; otherwise let AQE decide or tune join strategy from measured plans. Joining a large fact to a small dimension **without** the broadcast hint (when AQE misses it) forces an unnecessary shuffle of the large side.

### Partitions, repartition/coalesce, and pruning

- Model partitioning by **access pattern** (common filters) using **low-cardinality columns like date** — never by high-cardinality columns (that explodes into millions of tiny files).
- Use `repartition(n, cols...)` to increase/redistribute parallelism before heavy shuffle writes.
- Use `coalesce(n)` to reduce partitions cheaply (no full shuffle) before outputting files.
- Keep predicate columns explicit in filters to benefit from partition pruning and data skipping.

```python
# Bad: large event table written unpartitioned
df.write.format("delta").save("/mnt/events")

# Good: partition by a low-cardinality access-pattern column
df.write.format("delta") \
    .partitionBy("date") \
    .save("/mnt/events")
```

### Cache / persist / unpersist discipline

```python
from pyspark import StorageLevel

base = spark.table("prod.sales.orders").filter("order_date >= '2026-01-01'")
cached = base.persist(StorageLevel.MEMORY_AND_DISK)

cached.count()  # materialize cache once
# ... reuse cached in multiple downstream actions ...
cached.unpersist()
```

Cache only when a DataFrame is reused by multiple expensive actions; always `unpersist()` when done to avoid memory pressure on executors.

### Data skew and AQE skew join mitigation

Skew appears when a few join/group keys hold disproportionate rows, creating straggler tasks.

- Detect with `explain("formatted")`, Spark UI stage/task duration spread, and key frequency checks.
- Enable AQE and skew handling (`spark.sql.adaptive.enabled=true`, `spark.sql.adaptive.skewJoin.enabled=true`).
- Consider salting hot keys, pre-aggregating before wide joins, or splitting extreme keys into dedicated paths.

Reliability rule: optimize with measurements (Spark UI + query plans), not assumptions.

### Configurations (AQE, shuffle, skew, broadcast)

Tune configuration by workload and cluster size; avoid one-size-fits-all constants.

```python
# Adaptive Query Execution baseline
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Shuffle partitions: start from cluster parallelism, then tune with stage metrics
spark.conf.set("spark.sql.shuffle.partitions", "auto")  # or explicit integer after benchmarking

# Broadcast threshold: keep default unless plans show clear missed opportunities
# spark.conf.set("spark.sql.autoBroadcastJoinThreshold", 10485760)  # 10MB example, tune in-cluster
```

Re-check physical plans (`explain("formatted")`) after changing AQE/shuffle/broadcast settings.

## Schemas

### Why explicit schemas matter

`inferSchema` is convenient for exploration but expensive and risky for production pipelines:

- It may require an extra pass over input files before real processing.
- Type inference can drift when source payload quality changes (for example numeric-looking strings, null-heavy columns, mixed types).
- Inference inconsistencies can cascade into downstream failures or silent casts.

Use explicit `StructType` for ingestion paths that feed jobs/SLAs.

```python
from pyspark.sql.types import (
    StructType,
    StructField,
    StringType,
    TimestampType,
    DecimalType,
)

orders_schema = StructType([
    StructField("order_id", StringType(), False),
    StructField("customer_id", StringType(), True),
    StructField("order_ts", TimestampType(), True),
    StructField("amount", DecimalType(18, 2), True),
    StructField("status", StringType(), True),
])

orders_df = (
    spark.read
    .format("json")
    .schema(orders_schema)
    .load("s3://company-landing/orders/")
)
```

For ad-hoc notebooks, `inferSchema` is acceptable as a starting point, then lock the schema once data contracts stabilize.

### Delta schema note

Delta stores table metadata (including schema versions and evolution history) in the transaction log (`_delta_log`), which is why time travel and schema-aware reads stay reliable across commits. Prefer controlled evolution (`mergeSchema`, `overwriteSchema`) with review instead of implicit drift. See `delta.md` for table DDL and maintenance.

## Higher-order functions (arrays/maps)

Higher-order functions keep logic inside Spark SQL execution plans (better optimization than Python row-by-row code). Use them for nested/array-heavy payloads (JSON events, clickstream attributes, product lists) before reaching for UDFs.

```python
from pyspark.sql import functions as F

df = spark.createDataFrame(
    [
        ("o1", [10.0, 20.0, 30.0], ["ok", "vip"]),
        ("o2", [5.0, 7.5], ["fraud_check"]),
    ],
    ["order_id", "amounts", "tags"],
)

# transform: apply function element-wise
df_transform = df.withColumn("amounts_with_tax", F.transform("amounts", lambda x: x * F.lit(1.21)))

# filter: keep only elements satisfying predicate
df_filter = df.withColumn("large_amounts", F.filter("amounts", lambda x: x >= F.lit(10.0)))

# aggregate: reduce array to a single value (sum in this case)
df_aggregate = df.withColumn(
    "total_amount",
    F.aggregate("amounts", F.lit(0.0), lambda acc, x: acc + x),
)

# exists: at least one element matches
df_exists = df.withColumn("has_vip_tag", F.exists("tags", lambda t: t == F.lit("vip")))

# forall: all elements match
df_forall = df.withColumn("all_positive", F.forall("amounts", lambda x: x > F.lit(0.0)))
```

Equivalent SQL style:

```sql
SELECT
  order_id,
  transform(amounts, x -> x * 1.21) AS amounts_with_tax,
  filter(amounts, x -> x >= 10.0) AS large_amounts,
  aggregate(amounts, 0D, (acc, x) -> acc + x) AS total_amount,
  exists(tags, t -> t = 'vip') AS has_vip_tag,
  forall(amounts, x -> x > 0) AS all_positive
FROM prod.silver.orders_array_example;
```

## Databricks Jobs and Workflows

**Job configuration:**
```python
# Create job via API
job_config = {
    "name": "daily_etl_pipeline",
    "max_concurrent_runs": 1,
    "timeout_seconds": 3600,
    "schedule": {
        "quartz_cron_expression": "0 0 2 * * ?",
        "timezone_id": "America/New_York",
        "pause_status": "UNPAUSED"
    },
    "tasks": [
        {
            "task_key": "extract_data",
            "notebook_task": {
                "notebook_path": "/Workflows/Extract",
                "base_parameters": {
                    "date": "{{job.start_time.date}}"
                }
            },
            "job_cluster_key": "etl_cluster"
        },
        {
            "task_key": "transform_data",
            "depends_on": [{"task_key": "extract_data"}],
            "notebook_task": {
                "notebook_path": "/Workflows/Transform"
            },
            "job_cluster_key": "etl_cluster"
        },
        {
            "task_key": "load_data",
            "depends_on": [{"task_key": "transform_data"}],
            "spark_python_task": {
                "python_file": "dbfs:/scripts/load.py",
                "parameters": ["--env", "production"]
            },
            "job_cluster_key": "etl_cluster"
        }
    ],
    "job_clusters": [
        {
            "job_cluster_key": "etl_cluster",
            "new_cluster": {
                "spark_version": "13.3.x-scala2.12",
                "node_type_id": "i3.xlarge",
                "num_workers": 4
            }
        }
    ],
    "email_notifications": {
        "on_failure": ["data-eng@company.com"],
        "on_success": ["data-eng@company.com"]
    }
}
```

**dbutils cookbook (preloaded in Databricks notebooks):**

`dbutils` is always available in the notebook runtime (no import). Use it for filesystem I/O, parameters, secrets, and notebook composition.

```python
dbutils.help()  # overview of modules

# --- Filesystem (DBFS, Unity Catalog volumes, mounted paths) ---
dbutils.fs.ls("/path/")
dbutils.fs.cp("dbfs:/source/file", "dbfs:/dest/file")
dbutils.fs.mv("dbfs:/source/file", "dbfs:/dest/file")
dbutils.fs.rm("dbfs:/path/temp", recurse=True)
dbutils.fs.mkdirs("dbfs:/path/new_folder")
dbutils.fs.head("dbfs:/path/file.csv", maxBytes=1000)

# --- Widgets: interactive parameters without editing code ---
dbutils.widgets.text("nombre", "default_value", "Etiqueta")
dbutils.widgets.dropdown("pais", "AR", ["AR", "MX", "CO"], "País")
valor = dbutils.widgets.get("nombre")
dbutils.widgets.remove("nombre")
dbutils.widgets.removeAll()

# --- Secrets (configure scopes in workspace admin / secret scopes) ---
password = dbutils.secrets.get(scope="mi-scope", key="db-password")

# --- Notebook composition ---
result = dbutils.notebook.run("/path/to/other_notebook", timeout_seconds=60, arguments={"fecha": "2024-01-15"})
dbutils.notebook.exit("valor_de_retorno")  # return value to caller (job task or parent notebook)
```

**Pattern — dropdown to filter a dataset or environment:**

```python
dbutils.widgets.dropdown(
    "dataset",
    "wine-quality",
    ["wine-quality", "nyctaxi", "covid"],
    "Seleccionar dataset",
)
chosen = dbutils.widgets.get("dataset")
# use `chosen` to switch paths, catalog names, or SQL filters
```

**Good practice:** Prefer **widgets + secrets** over hardcoding passwords or full paths. For production jobs, pass the same values via **job parameters** / task parameters and read them with `dbutils.widgets` or task APIs so notebooks stay portable. Run production jobs as **service principals**, not personal users (see `unity-catalog.md`).
