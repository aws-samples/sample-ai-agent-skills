# AWS HealthImaging Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any HealthImaging issue.

## Guardrail 1: Datastores Are Isolated Containers
Each datastore is an independent container for medical imaging data. Image sets in one datastore cannot be accessed from another without a copy operation. Datastore IDs are region-specific and globally unique.

## Guardrail 2: DICOM Import Jobs Can Partially Succeed
Import jobs process files individually. Some files may succeed while others fail DICOM validation. A job status of COMPLETED does not mean all files were imported — always check the output manifest for per-file results and error details.

## Guardrail 3: Image Sets Are Immutable and Versioned
Image sets use immutable versioning. Each update (adding frames, updating metadata) creates a new version. Previous versions are retained. Do not assume in-place modification — always check the latest version ID.

## Guardrail 4: Metadata Search Is Limited to Indexed Tags
Only specific DICOM tags at patient, study, and series levels are indexed for search. Instance-level tags and custom private tags are NOT searchable via search-image-sets. Retrieve full metadata with get-image-set-metadata for complete tag access.

## Guardrail 5: Encryption Key Cannot Be Changed After Creation
Datastore encryption (AWS-managed or customer-managed KMS key) is set at creation time and cannot be modified. To change encryption, you must create a new datastore with the desired key and copy image sets. Never suggest changing encryption on an existing datastore.

## Guardrail 6: Deletion Has a Retention Window
Deleted image sets may be recoverable within a retention period. Permanent deletion (purge) is a separate operation. Do not assume deletion is immediate and irreversible unless purge has been confirmed.

## Guardrail 7: Copy Operations Are Asynchronous
Copy jobs between datastores run asynchronously and may take significant time for large image sets. The destination copy is independent — subsequent changes to the source do not propagate. Always verify copy job completion status.

## Guardrail 8: Image Frame Retrieval Returns HTJ2K Format
Image frames are returned in High-Throughput JPEG 2000 (HTJ2K) format, not raw DICOM pixel data. Client applications must support HTJ2K decoding. Do not assume standard JPEG or raw pixel data output.

## Guardrail 9: IAM Permissions Are Datastore-Scoped
IAM policies can restrict access to specific datastores using resource ARNs. A user with permissions on one datastore may not have access to another. Always verify permissions against the specific datastore ARN.

## Guardrail 10: S3 Input Bucket Must Be in Same Region
DICOM import jobs require the source S3 bucket to be in the same AWS region as the datastore. Cross-region imports are not supported. Verify bucket and datastore region alignment before troubleshooting import failures.

## Guardrail 11: DICOM Compliance Is Strictly Enforced
HealthImaging validates DICOM compliance during import. Files missing required DICOM tags (Patient ID, Study Instance UID, Series Instance UID, SOP Instance UID) are rejected. Non-standard DICOM files from legacy systems may need preprocessing.

## Guardrail 12: Service Quotas Limit Concurrent Operations
HealthImaging has service quotas for concurrent import jobs, datastores per account, and API request rates. Throttling errors may indicate quota limits, not service failures. Check Service Quotas before assuming a bug.
