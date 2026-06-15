---
name: azure-fabric-using-spark-job-definition
description: Spark Job Definition (SJD) in Microsoft Fabric — running headless Spark batch jobs (PySpark/Scala/SparkR) on OneLake, scheduling, CI/CD.
triggers: azure, azure fabric using spark job definition, fabric, onelake, sjd, jar, cli, api, spark job definition, microsoft fabric, spark, pyspark, scala, java jar, sparkr, git
license: MIT
version: 1.0.0
updated: 2026-06-04
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Spark Job Definition in Microsoft Fabric

A **Spark Job Definition (SJD)** is a Fabric workspace item that runs a **headless,
non-interactive Spark batch job** — a `.py` script, an executable Python file, or a
compiled `.jar` (Scala/Java) — without a notebook session. Use it when you want
production batch jobs that are versionable, parameterized, scheduled, and orchestrated
from pipelines, rather than interactive exploration in a notebook.

> **When to use SJD vs Notebook**
> - **Notebook** → interactive development, ad-hoc analysis, visualization, mixed-language cells.
> - **SJD** → packaged, repeatable batch jobs; JAR-based Scala/Java workloads; CI/CD-managed scripts; jobs triggered by schedule or pipeline activity.

---

## 1. Architecture & Item Model

```
Workspace
└── Spark Job Definition (item)
    ├── Main definition file        → main.py | app.jar  (the entry point)
    ├── Reference / auxiliary files  → .py, .jar, .zip, .whl helpers on the path
    ├── Command-line arguments       → positional/flag args passed to the job
    ├── Default lakehouse + others   → mounted OneLake context (abfss path roots)
    ├── Spark environment binding     → Environment item (libs + Spark config)
    └── Language: PySpark | Scala/Java | SparkR
```

Each run launches its **own Spark application** on the workspace capacity's Spark pool.
SJD runs draw from the same **Spark vCore allocation** governed by the capacity SKU
(see the `azure-fabric` skill's CU/vCore table) and are subject to **job queueing**
and **bursting/smoothing** like notebook jobs.

The main file and reference files live inside the SJD item in OneLake; you can also
point the main definition file at a file already stored in a connected lakehouse.

---

## 2. Supported Job Types

| Language | Main file | Reference files | Notes |
|----------|-----------|-----------------|-------|
| **PySpark** | `main.py` | `.py`, `.zip`, `.whl`, `.jar` | Most common; bind libraries via Environment or `.whl`/`.zip` |
| **Scala / Java** | `app.jar` | additional `.jar` | Must specify the **main class** in command-line args/config |
| **SparkR** | `main.R` | `.R`, `.tar.gz` | R batch workloads |

The main definition file must be an executable entry point. For JAR jobs, supply the
fully-qualified main class (e.g. `com.contoso.etl.DailyLoad`) so Spark knows where to start.

---

## 3. Creating an SJD (Portal)

1. Workspace → **+ New** → **Spark Job Definition** (under Data Engineering).
2. Name it, then in the SJD editor set:
   - **Language** (PySpark / Scala-Java / SparkR).
   - **Main definition file** — upload or browse to a lakehouse file.
   - **Reference files** — supporting modules/jars.
   - **Command line arguments** — space-separated args read by your job.
   - **Default lakehouse** (+ optional additional lakehouses) — provides the OneLake
     mount and a default `spark.sql` catalog context.
   - **Environment** — bind a Fabric **Environment** item for libraries + Spark config.
3. **Run** to launch immediately, or attach a schedule / use it in a pipeline.

---

## 4. Authoring the Job (PySpark example)

A self-contained `main.py` that reads args, writes a Delta table to the default lakehouse:

```python
import sys
import argparse
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

def parse_args(argv):
    p = argparse.ArgumentParser()
    p.add_argument("--source_path", required=True)
    p.add_argument("--target_table", required=True)
    p.add_argument("--run_date", required=True)
    return p.parse_args(argv)

def main():
    args = parse_args(sys.argv[1:])
    spark = SparkSession.builder.appName("daily-load").getOrCreate()

    df = (spark.read.format("parquet").load(args.source_path)
          .withColumn("ingest_date", F.lit(args.run_date)))

    # Default lakehouse is mounted; write as a managed Delta table
    (df.write.format("delta")
       .mode("overwrite")
       .option("overwriteSchema", "true")
       .saveAsTable(args.target_table))

    print(f"Wrote {df.count()} rows to {args.target_table} for {args.run_date}")

if __name__ == "__main__":
    main()
```

**Command line arguments** field (in the SJD editor):

```
--source_path abfss://Sales@onelake.dfs.fabric.microsoft.com/Bronze.Lakehouse/Files/raw/2026-05-28 --target_table silver_orders --run_date 2026-05-28
```

> **Tip** — Don't hardcode dates/paths in the script. Read everything from
> command-line args so a pipeline can supply them dynamically (e.g. `@utcNow()`).

### Scala/Java JAR job

```scala
package com.contoso.etl
import org.apache.spark.sql.SparkSession

object DailyLoad {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder.appName("daily-load").getOrCreate()
    val Array(sourcePath, targetTable, runDate) = args
    val df = spark.read.parquet(sourcePath)
    df.write.format("delta").mode("overwrite").saveAsTable(targetTable)
    spark.stop()
  }
}
```

Build a JAR, upload as the **main definition file**, set the main class, and pass the
three positional args in the command-line field.

---

## 5. OneLake / Lakehouse Access Patterns

- A **default lakehouse** gives you `spark.read.table(...)` / `saveAsTable(...)`
  against the default catalog, plus a mounted Files root.
- For any OneLake location (including other workspaces' items), use the full **abfss**
  path:

  ```
  abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<item>.Lakehouse/Tables/<table>
  abfss://<workspace>@onelake.dfs.fabric.microsoft.com/<item>.Lakehouse/Files/<path>
  ```

- The job runs under the **identity context** of the run owner (manual) or the
  scheduling/pipeline identity; ensure that identity has access to all referenced items.

---

## 6. Environments & Libraries

Bind a Fabric **Environment** item to the SJD to control:
- **Spark compute** — pool size, dynamic allocation, runtime version.
- **Spark properties** — e.g. `spark.sql.shuffle.partitions`, custom configs.
- **Libraries** — public (PyPI/conda) and custom (`.whl`, `.jar`, `.tar.gz`).

If you don't bind an Environment, the SJD uses the **workspace default Spark settings**.
For self-contained dependency packaging without an Environment, attach `.whl`/`.zip`
(Python) or extra `.jar` files (Scala) as **reference files**.

> **Gotcha** — Publishing an Environment can take several minutes; the SJD picks up the
> published version, not draft changes. Re-publish after library edits.

---

## 7. Scheduling & Orchestration

### Built-in schedule
SJD editor → **Settings → Schedule** → set cadence (by the minute / hourly / daily /
weekly) and time zone. Good for simple recurring batch jobs.

### Pipeline activity (recommended for orchestration)
Add a **Spark Job Definition activity** to a Data Factory pipeline to:
- Pass **dynamic parameters** as command-line args (`@formatDateTime(utcNow(),'yyyy-MM-dd')`).
- Chain with copy activities, lookups, conditional logic, and notebook activities.
- Centralize retries, alerting, and dependency ordering.

The activity returns run status; downstream activities can branch on success/failure.

---

## 8. Monitoring & Debugging

- **Run history** — SJD item → **Recent runs**, or the workspace **Monitoring hub**
  filtered to Spark Job Definition.
- **Spark UI / Spark application detail** — drill into stages, executors, and logs
  per run from the run's detail page.
- **Logs** — driver stdout/stderr (your `print` statements appear there); access via
  the application detail / log download.
- **Snapshot** — each run captures the main file + args + config used, aiding repro.

Common failure causes:
| Symptom | Likely cause |
|---------|-------------|
| Job queued forever | Capacity Spark vCores exhausted / throttling — check capacity metrics app |
| `ModuleNotFoundError` | Library not in bound Environment or missing reference `.whl` |
| `ClassNotFoundException` | Wrong/missing main class for JAR job |
| `Path does not exist` (abfss) | Identity lacks access to the lakehouse/workspace, or wrong path |
| `Table not found` via `saveAsTable` | No default lakehouse bound |

---

## 9. CI/CD — Git Integration & APIs

### Git integration
SJD items are **Git-integration supported**. When you connect a workspace to Azure
DevOps / GitHub, an SJD serializes to a folder containing its definition (platform
metadata + properties); the main/reference files are referenced. Commit, branch, and
sync via the workspace **Source control** panel.

### REST API (create / run)
The Fabric REST API exposes Spark Job Definitions under the items endpoint:

```bash
# Create an SJD item (definition payload is base64-encoded parts)
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/sparkJobDefinitions

# Trigger an on-demand run via the job scheduler
POST https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/sparkJobDefinitions/{sjdId}/jobs/instances?jobType=sparkjob

# Poll run status
GET  https://api.fabric.microsoft.com/v1/workspaces/{workspaceId}/items/{sjdId}/jobs/instances/{instanceId}
```

Authenticate with a bearer token (`https://api.fabric.microsoft.com/.default` scope)
for a service principal or user. The same operations are available through the
**Fabric CLI (`fab`)** and the `fabric-cicd` deployment tooling for promoting SJDs
across dev/test/prod workspaces.

---

## 10. Best Practices

- **Parameterize everything** — paths, dates, table names via command-line args, not literals.
- **Keep the main file thin** — push reusable logic into a `.whl`/module shipped as a
  reference file or Environment library; version it.
- **Pin a runtime + Environment** for reproducibility; don't rely on the moving workspace default.
- **One job, one responsibility** — small composable SJDs orchestrated by a pipeline beat
  one monolithic script.
- **Idempotent writes** — use `overwrite` with partition filters or Delta `MERGE` so
  re-runs after failure are safe.
- **Right-size the pool** — batch jobs benefit from larger executors; tune via the bound
  Environment rather than per-script `spark.conf`.
- **Use the Monitoring hub + capacity metrics** to catch queueing before SLAs slip.

---

## Sources

Grounded in Microsoft Learn documentation for Fabric Data Engineering — Spark Job
Definition (`learn.microsoft.com/en-us/fabric/data-engineering/spark-job-definition`,
`…/create-spark-job-definition`, `…/run-spark-job-definition`, `…/spark-job-definition-source-control`)
and the Fabric REST API / Fabric CLI references. Always verify SKU- and preview-gated
behavior against current docs, as Fabric evolves rapidly.
