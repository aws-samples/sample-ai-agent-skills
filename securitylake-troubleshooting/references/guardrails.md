# Amazon Security Lake Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Security Lake issue.

## Guardrail 1: Lake Formation Controls Data Access, Not S3 Policies
Security Lake data access is governed by AWS Lake Formation permissions, not S3 bucket policies. Subscribers need Lake Formation grants on the Glue database and tables. Adding S3 bucket policy permissions alone does NOT grant query access. Always check Lake Formation permissions first.

## Guardrail 2: OCSF Schema Is Mandatory for All Sources
All data in Security Lake must conform to OCSF (Open Cybersecurity Schema Framework). Native AWS sources are automatically normalized. Custom sources MUST deliver data in OCSF-compliant Parquet format. Non-compliant data is rejected. Validate custom source schema before troubleshooting delivery.

## Guardrail 3: Cross-Account Requires Organizations Delegated Admin
Cross-account log collection requires AWS Organizations. A delegated administrator account must be designated to manage Security Lake across the organization. Individual account-to-account sharing is not supported. The delegated admin configures sources for member accounts.

## Guardrail 4: Rollup Regions Copy Data, Not Move
Rollup regions aggregate data by COPYING from contributing regions to the rollup region. Source regions retain their data independently. Both regions have data and incur storage costs. Deleting data in the rollup region does not affect source regions.

## Guardrail 5: Subscriber Types Have Different Access Patterns
Query subscribers get Lake Formation permissions to query data via Athena/Glue. Data access subscribers receive SQS notifications or EventBridge events when new data arrives and pull from S3 directly. The subscriber type determines the access mechanism. Do not mix up the two.

## Guardrail 6: Security Lake Manages the S3 Bucket
The S3 bucket is created and managed by Security Lake. Do not modify bucket policies, lifecycle rules, or encryption settings directly. Use Security Lake APIs to configure retention and access. Direct bucket modifications may break Security Lake functionality.

## Guardrail 7: Data Appears with Ingestion Delay
Log sources have ingestion delays. CloudTrail management events: ~5-15 minutes. VPC Flow Logs: ~10-15 minutes. Custom sources: depends on delivery frequency. Do not diagnose "missing data" until sufficient time has passed. Check `list-data-lake-exceptions` for actual errors.

## Guardrail 8: Glue Database and Tables Are Auto-Created
Security Lake automatically creates a Glue database (amazon_security_lake_glue_db_<region>) and tables for each source. Do not manually create or modify these tables. Schema changes from Security Lake updates are applied automatically. Manual table modifications break queries.

## Guardrail 9: Custom Sources Need Explicit Registration
Custom sources must be registered via `create-custom-log-source` before delivering data. Registration creates the S3 prefix, Glue table, and crawler. Data delivered without registration is not processed. Each custom source gets a unique source name and version.

## Guardrail 10: Retention Applies Per Source and Region
Data retention (expiration) is configured per source type and per region. Different sources can have different retention periods. Retention is implemented via S3 lifecycle rules managed by Security Lake. Do not set lifecycle rules directly on the bucket.

## Guardrail 11: Athena Queries Use Glue Catalog Tables
Athena queries against Security Lake data use the Glue Data Catalog tables, not direct S3 paths. The query must reference the correct database and table name. Table names include region and source type. Partition projection is used for efficient querying.

## Guardrail 12: Exceptions API Shows Ingestion Errors
Use `list-data-lake-exceptions` to check for source ingestion errors. This API returns specific error messages for failed ingestion attempts. Common exceptions include permission errors, schema validation failures, and S3 delivery issues. Always check exceptions before assuming data loss.
