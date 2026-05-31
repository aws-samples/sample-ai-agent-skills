# Amazon Timestream Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Timestream issue.

## Guardrail 1: Memory Store Retention Is in HOURS, Magnetic Store Is in DAYS
Memory store retention ranges from 1 to 8760 hours. Magnetic store retention ranges from 1 to 73000 days. These are different units. Setting memory store retention to "24" means 24 hours, not 24 days. Setting magnetic store retention to "365" means 365 days. Always verify the unit when configuring retention.

## Guardrail 2: Late-Arriving Data Is Rejected by Default
Records with timestamps older than the memory store retention window are rejected unless magnetic store writes are enabled on the table. This is a common cause of "RejectedRecords" errors. Enable magnetic store writes to accept late-arriving data: `aws timestream-write update-table --database-name <db> --table-name <table> --magnetic-store-write-properties EnableMagneticStoreWrites=true`.

## Guardrail 3: Data Is Append-Only — No UPDATE or DELETE
Timestream does not support UPDATE or DELETE operations on individual records. Data is immutable once written. The only way to remove data is to let it expire via retention policies. If incorrect data is written, you can write corrected records with the same dimensions and time but different measure values — the latest version wins for queries.

## Guardrail 4: Multi-Measure vs Single-Measure Records
Multi-measure records store up to 256 measures per record and are more storage-efficient. Single-measure records store one measure per record. Mixing both types in the same table is not recommended and can cause query complexity. Choose one approach per table and stick with it.

## Guardrail 5: Scheduled Queries Require IAM Role, SNS, and S3
Scheduled queries need: (1) an IAM role with permissions to read from source and write to destination, (2) an SNS topic for execution notifications, and (3) an S3 bucket for error reports. Without SNS configuration, scheduled query failures are silent. Always configure SNS notifications.

## Guardrail 6: Dimensions Are Indexed — Choose Carefully
Dimensions are automatically indexed and used for query filtering. They cannot be changed after data is written. High-cardinality dimensions (like unique request IDs) cause performance issues and increased costs. Use dimensions for attributes you'll filter on frequently. Use measures for values you'll aggregate.

## Guardrail 7: Timestream for InfluxDB Is a Different Service
Timestream for InfluxDB and Timestream for LiveAnalytics are completely different services with different APIs, pricing models, query languages, and capabilities. Timestream for LiveAnalytics uses SQL. Timestream for InfluxDB uses InfluxQL/Flux. Do not confuse CLI commands or documentation between the two.

## Guardrail 8: Endpoint Discovery Is Required
Timestream uses a cell-level architecture requiring endpoint discovery. Clients must call DescribeEndpoints to get the correct write and query endpoints. The AWS SDK handles this automatically, but custom HTTP clients must implement endpoint discovery. Endpoints can change — cache them but refresh periodically.

## Guardrail 9: Magnetic Store Writes Must Be Explicitly Enabled
Magnetic store writes are disabled by default on new tables. Data can only be written to the memory store unless magnetic store writes are explicitly enabled. Late-arriving data that falls outside the memory store window is rejected without this setting. Check with describe-table and look for MagneticStoreWriteProperties.

## Guardrail 10: Query Results Are Paginated
Timestream query results are paginated. Large result sets require handling pagination tokens. The maximum number of rows per page varies. Always check for NextToken in query responses and continue fetching until no more pages remain. The SDK handles pagination automatically with paginators.

## Guardrail 11: Data Model Cannot Be Changed After Writing
Once data is written with specific dimensions and measure names, the schema is established. You cannot rename dimensions, change measure types, or restructure existing data. Plan your data model carefully before writing production data. Test with a development table first.

## Guardrail 12: CloudWatch Metrics Distinguish System vs User Errors
Timestream publishes SystemErrors (AWS-side issues) and UserErrors (client-side issues like invalid queries or rejected records) to CloudWatch. High UserErrors indicate application-side problems. High SystemErrors indicate service-side issues. Always check both metrics when diagnosing problems.
