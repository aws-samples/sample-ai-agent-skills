# Amazon Athena Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Athena issue.

## Guardrail 1: Athena Charges Per TB of Data Scanned
Athena pricing is based on the amount of data scanned by each query — not per query count, not per compute time, not per row returned. Using columnar formats (Parquet, ORC) reduces data scanned because Athena reads only the needed columns. Partitioning reduces data scanned by skipping irrelevant S3 prefixes. Compression further reduces scanned bytes. A full scan of unpartitioned CSV data is the worst-case cost scenario. Do NOT tell users Athena is free or has flat-rate pricing (unless they use provisioned capacity).

## Guardrail 2: Default Query Timeout Is 30 Minutes
Each Athena query has a default timeout of 30 minutes. This is configurable per workgroup up to the service maximum. Queries exceeding the timeout are automatically cancelled with a FAILED status. If a query consistently times out, the fix is query optimization (partitioning, columnar format, predicate pushdown) — not just increasing the timeout. Check `get-query-execution` for the actual runtime before assuming timeout.

## Guardrail 3: Workgroups Control Cost, Access, and Configuration
Workgroups are the primary control mechanism in Athena. Each workgroup has its own: query result S3 location, encryption settings, per-query data scan limit (bytes), engine version, and IAM-based access control. Users inherit workgroup settings and cannot override them if the workgroup enforces configuration. Always check the workgroup configuration when debugging unexpected behavior.

## Guardrail 4: Glue Data Catalog Is the Metastore
Athena uses AWS Glue Data Catalog as its metastore for databases, tables, and partitions. Table definitions (schema, SerDe, input format, S3 location) are stored in Glue. If the Glue table definition is wrong, Athena queries fail or return incorrect results. Always verify the Glue table definition with `aws glue get-table` before investigating query logic.

## Guardrail 5: CTAS and INSERT INTO Are for ETL
CREATE TABLE AS SELECT (CTAS) creates a new table from query results in a specified format and location. INSERT INTO appends query results to an existing table. Both write data to S3. CTAS is commonly used to convert CSV to Parquet, create aggregated tables, or materialize query results. These are batch operations — not streaming or real-time.

## Guardrail 6: Federated Queries Use Lambda Connectors
Athena federated queries use Lambda-based data source connectors to query external systems (RDS, DynamoDB, Redis, Elasticsearch, etc.). The Lambda function must have: VPC access if the source is in a VPC, IAM permissions to read the source, and sufficient memory/timeout configuration. Federated queries are inherently slower than native S3 queries due to Lambda invocation overhead and network latency.

## Guardrail 7: Query Result Location Must Be Writable
Every Athena query writes results to an S3 location specified in the workgroup or query configuration. The executing principal must have s3:PutObject, s3:GetBucketLocation, and s3:AbortMultipartUpload on the result bucket/prefix. If the result location is not writable, the query fails AFTER execution — wasting compute. Always verify result location permissions separately from data source permissions.

## Guardrail 8: Engine v3 (Trino) vs Engine v2 (Presto) Differences
Athena engine v3 is based on Trino and engine v2 is based on Presto. There are syntax differences: v3 supports MERGE, improved TIMESTAMP handling, and different function signatures. Queries that work on v2 may fail on v3 and vice versa. Always check the workgroup engine version with `get-work-group` when debugging syntax errors. Do NOT assume all Athena environments use the same engine.

## Guardrail 9: Partition Projection Computes Partitions On-the-Fly
Partition projection eliminates the need for MSCK REPAIR TABLE or Glue crawlers by defining partition patterns in table properties. It requires correct configuration of projection type (enum, integer, date, injected), range, format, and interval. Wrong projection config causes empty results — not errors. Always verify projection properties in the Glue table definition when partitioned queries return no data.

## Guardrail 10: SerDe Must Match the Actual Data Format
The Serializer/Deserializer (SerDe) in the Glue table definition must match the actual file format in S3. OpenCSVSerDe for CSV, JsonSerDe or HiveJsonSerDe for JSON, ParquetHiveSerDe for Parquet, OrcSerDe for ORC, LazySimpleSerDe for TSV/delimited. Using the wrong SerDe produces NULL values, parse errors, or HIVE_BAD_DATA exceptions. Always check both the SerDe and the actual S3 file format.

## Guardrail 11: S3 Path Cleanliness Matters
Athena reads ALL files under the table's S3 location prefix. Stray files like _SUCCESS markers, .crc files, hidden files (._*), or files in a different format cause query failures or incorrect results. Use S3 lifecycle rules or explicit cleanup to keep table locations clean. The IGNORE_MALFORMED_DATA property can skip bad records but masks real problems.

## Guardrail 12: No UPDATE/DELETE on Regular Tables
Athena does not support UPDATE or DELETE statements on regular (Hive) tables. For mutable data operations, use Apache Iceberg table format which supports ACID transactions (MERGE, UPDATE, DELETE) in Athena engine v3. Do NOT suggest UPDATE/DELETE on non-Iceberg tables — it will fail with a syntax error.
