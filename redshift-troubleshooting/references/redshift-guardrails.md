# Amazon Redshift Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Redshift issue.

## Guardrail 1: Leader Node vs Compute Nodes Have Different Roles
The leader node handles SQL parsing, query planning, query optimization, and final result aggregation. Compute nodes store data slices and execute query fragments in parallel. Queries that reference no user tables (e.g., SELECT 1, SELECT CURRENT_DATE) run entirely on the leader node. System tables prefixed with STL_, SVL_, STV_ reside on the leader. If a query is "leader-node only," adding compute nodes won't help.

## Guardrail 2: WLM Controls Query Concurrency — Not Infinite
Workload Management (WLM) queues control how many queries run concurrently and how much memory each gets. Default manual WLM allows 5 concurrent queries per queue. Auto WLM dynamically adjusts concurrency (recommended). Queries exceeding the concurrency limit wait in the queue — this is NOT a bug, it's by design. Check stv_wlm_query_state for queue wait times before blaming query performance.

## Guardrail 3: COPY Is the Only Efficient Bulk Load Method
COPY loads data from S3, DynamoDB, EMR, or remote hosts in parallel across all compute nodes. Single-row INSERT statements are extremely slow because each is a separate transaction with commit overhead. Multi-row INSERT (INSERT INTO ... VALUES) is slightly better but still far slower than COPY. Always recommend COPY for any bulk data loading. The only exception is small ad-hoc inserts of a few rows.

## Guardrail 4: Distribution Keys Determine Data Placement
The distribution style (KEY, EVEN, ALL, AUTO) determines how rows are distributed across compute node slices. Wrong distribution keys cause: (1) data skew — one slice has far more rows than others, (2) redistribution — data must be moved between nodes during joins. Check svv_table_info for skew_rows. A skew_rows value > 4.0 indicates significant skew. Do NOT change distribution style without analyzing join patterns first.

## Guardrail 5: Sort Keys Require VACUUM Maintenance
Sort keys determine the physical order of data on disk. Unsorted data (from recent loads) is stored in an unsorted region and requires a full table scan. VACUUM SORT re-sorts the unsorted region. Check svv_table_info "unsorted" column — values > 5% degrade scan performance. Compound sort keys benefit queries filtering on leading columns. Interleaved sort keys help varied filter patterns but have higher VACUUM cost.

## Guardrail 6: VACUUM Is Still Required Despite Auto-VACUUM
Redshift auto-VACUUM handles DELETE reclamation (VACUUM DELETE) automatically in the background. However, VACUUM SORT (re-sorting unsorted rows) may still need manual execution for large tables with heavy insert activity. Auto-VACUUM does NOT run VACUUM FULL. Check the "unsorted" and "empty" percentages in svv_table_info. Do NOT tell users VACUUM is never needed.

## Guardrail 7: Spectrum Queries S3 In-Place — No Data Loading
Redshift Spectrum uses external tables to query data in S3 directly. Data is NOT loaded into Redshift storage. Spectrum uses compute nodes to process S3 data, and the leader node handles the final aggregation. Performance depends on: file format (Parquet/ORC columnar formats are 10x faster than CSV), file size (100MB-1GB optimal), partition pruning, and predicate pushdown. Do NOT suggest Spectrum "loads" data.

## Guardrail 8: Concurrency Scaling Is Read-Only
Concurrency scaling adds transient clusters to handle burst read query workloads. It does NOT handle write operations (INSERT, UPDATE, DELETE, COPY, UNLOAD). Concurrency scaling activates automatically when WLM queues have waiting queries and the feature is enabled. Billing is per-second of concurrency scaling cluster usage. Each cluster gets 1 hour of free concurrency scaling credits per day per main cluster.

## Guardrail 9: Serverless Uses RPUs — Not Nodes
Redshift Serverless uses Redshift Processing Units (RPUs) instead of fixed node types. Base capacity is configurable (minimum 8 RPU, increments of 8). You pay for compute time in RPU-seconds, not for always-on nodes. There are no node types, no node counts, and no resize operations in Serverless. Performance scales with RPU allocation. Do NOT reference node types when discussing Serverless.

## Guardrail 10: Cross-Database Queries Are Read-Only
You can query across databases in the same Redshift cluster using three-part notation (database.schema.table). However, cross-database queries are READ-ONLY — you cannot INSERT, UPDATE, or DELETE across databases. External schemas (Spectrum, federated queries) cannot be accessed cross-database. Materialized views cannot reference cross-database objects.

## Guardrail 11: Maintenance Windows Cause Brief Unavailability
During scheduled maintenance, Redshift may restart the cluster to apply patches. This causes a brief period of unavailability (typically minutes). Deferred maintenance allows postponing non-critical patches for up to 45 days. Mandatory security patches cannot be deferred. Schedule maintenance windows during low-traffic periods. Do NOT tell users maintenance has zero downtime.

## Guardrail 12: RA3 Managed Storage Separates Compute and Storage
RA3 node types use managed storage that automatically scales. Data is cached on local SSDs and spills to S3. This means disk space is virtually unlimited for RA3 nodes — the PercentageDiskSpaceUsed metric reflects local cache usage, not a hard limit. For DC2 nodes, disk space IS a hard limit. Do NOT conflate RA3 and DC2 storage behavior.
