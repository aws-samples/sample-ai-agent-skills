# AWS HealthLake Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS HealthLake issue.

## Guardrail 1: HealthLake Uses FHIR R4 Standard
HealthLake supports FHIR R4 (Release 4) only. FHIR STU3 or earlier resources must be converted. Resource types, search parameters, and operations follow the FHIR R4 specification. Custom resource types are not supported.

## Guardrail 2: Data Stores Are Encrypted by Default
All data stores use KMS encryption. You can use AWS-managed keys or customer-managed KMS keys. The KMS key must be in the same region. Changing encryption after creation requires creating a new data store.

## Guardrail 3: Import Jobs Are Asynchronous
FHIR import jobs run asynchronously. Input must be in NDJSON format in S3. Each line is a FHIR Bundle or individual resource. Import jobs validate FHIR resources. Invalid resources are reported in the job output. Large imports can take hours.

## Guardrail 4: Export Jobs Write to S3
FHIR export jobs write NDJSON files to S3. The IAM role must have s3:PutObject permission. Export can filter by resource type and time range. Export jobs are asynchronous and can take significant time for large data stores.

## Guardrail 5: Search Uses FHIR Search Parameters
HealthLake search follows FHIR R4 search specification. Not all FHIR search parameters may be supported. Search results are paginated. Complex searches may have performance implications. Use _include and _revinclude for related resources.

## Guardrail 6: NLP Enrichment Is Automatic When Enabled
When integrated with Amazon Comprehend Medical, NLP enrichment automatically extracts medical entities from clinical notes. Enrichment adds extensions to FHIR resources. It is not a manual process. Enrichment adds processing time to imports.

## Guardrail 7: FHIR REST API Has Rate Limits
The FHIR REST API has per-data-store rate limits for read, write, and search operations. Throttling returns HTTP 429. Batch operations help reduce individual API calls. Request limit increases via Service Quotas.

## Guardrail 8: Data Store Creation Takes Time
Creating a data store is asynchronous and can take several minutes. The data store must be in ACTIVE status before use. Check status with describe-fhir-datastore. Failed creation may be due to KMS key issues or account limits.

## Guardrail 9: NDJSON Format Is Required for Import
Import data must be in NDJSON (Newline Delimited JSON) format. Each line must be a valid FHIR resource or Bundle. Regular JSON files will fail import. Files must be in S3 with proper IAM access.

## Guardrail 10: Analytics Integration Uses Lake Formation
HealthLake analytics integrates with Lake Formation for SQL-based analytics. Data is automatically exported to a Lake Formation database. Athena can query the analytics data. Analytics must be enabled on the data store.

## Guardrail 11: Access Control Uses IAM and SMART on FHIR
HealthLake supports IAM-based access control and SMART on FHIR authorization. IAM policies control API access. SMART on FHIR provides fine-grained access based on patient context. Both can be used together.

## Guardrail 12: Cross-Account Access Requires IAM Configuration
Accessing HealthLake from another account requires cross-account IAM role configuration. Resource policies are not supported. The data store KMS key must also grant cross-account access if using customer-managed keys.
