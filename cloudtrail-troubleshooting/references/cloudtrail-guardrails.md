# AWS CloudTrail Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AWS CloudTrail issue.

## Guardrail 1: Management Events vs Data Events Are Different
Management events (control plane) are API calls that manage AWS resources (CreateBucket, RunInstances). Data events (data plane) are operations on resources (GetObject, PutObject, Invoke). Management events are logged by default (first copy free). Data events must be explicitly enabled and are always paid.

## Guardrail 2: CloudTrail Log Delivery Has 5-15 Minute Latency
CloudTrail does not deliver logs in real-time. Logs are delivered to S3 approximately every 5-15 minutes. For near-real-time event processing, use CloudWatch Logs integration (1-2 minute delay) or EventBridge (near-real-time for supported events). Do not expect immediate log availability in S3.

## Guardrail 3: Organization Trails Are Managed Centrally
Organization trails must be created from the management account or delegated administrator. They log events for all member accounts. Member accounts can see the trail but cannot modify or delete it. Organization trails deliver logs to a central S3 bucket in the management/admin account.

## Guardrail 4: CloudTrail Lake and S3 Trails Are Separate Features
CloudTrail Lake stores events in event data stores with SQL query capability. S3 trails deliver JSON log files to S3 buckets. They have different APIs, pricing, and retention models. A trail can deliver to both S3 and Lake simultaneously. Lake queries use different syntax than S3-based log analysis.

## Guardrail 5: Log File Validation Detects Tampering
When enabled, CloudTrail creates SHA-256 digest files every hour. These digests can verify that log files have not been modified or deleted. Validation is performed using the validate-logs CLI command. Digest files are delivered to the same S3 bucket as logs. Disabling validation does not delete existing digest files.

## Guardrail 6: S3 Bucket Policy Must Allow CloudTrail Service Principal
The S3 bucket policy must grant s3:PutObject permission to the CloudTrail service principal (cloudtrail.amazonaws.com) and s3:GetBucketAcl to verify bucket ownership. Without these permissions, log delivery fails. The bucket must also exist in the same partition (aws, aws-cn, aws-us-gov).

## Guardrail 7: KMS Key Policy Must Allow CloudTrail Encrypt/Decrypt
When using SSE-KMS encryption, the KMS key policy must allow CloudTrail to GenerateDataKey for encryption. Users reading logs need kms:Decrypt permission. Missing key permissions cause delivery failures or inability to read logs. The key must be in the same region as the trail.

## Guardrail 8: Data Events Can Generate High Volume and Cost
S3 object-level logging can generate millions of events per day for active buckets. Lambda invocation logging generates an event per invocation. Use advanced event selectors to filter by specific buckets, prefixes, or functions. Monitor data event volume to control costs.

## Guardrail 9: Multi-Region Trails Capture Events in All Regions
A multi-region trail logs events from all AWS regions. A single-region trail only logs events in the region where it was created. Global service events (IAM, STS, CloudFront) are logged in us-east-1 by default. Multi-region trails capture these automatically.

## Guardrail 10: Advanced Event Selectors Replace Basic Event Selectors
Advanced event selectors provide more granular control than basic event selectors. They support filtering by resource ARN, event name, and read/write type. A trail uses either basic OR advanced selectors, not both. Switching from basic to advanced is one-way — you cannot revert.

## Guardrail 11: Insights Requires Sufficient Baseline Activity
CloudTrail Insights detects anomalies by comparing current API call rates against a baseline. The baseline requires 7 days of normal activity. Insights may not detect anomalies during the initial baseline period. Insights events are separate from management/data events and have their own pricing.

## Guardrail 12: Event History Retains 90 Days of Management Events
The CloudTrail Event History in the console shows the last 90 days of management events for free. This is available without creating a trail. For longer retention, create a trail to deliver to S3 or use CloudTrail Lake. Data events are NOT shown in Event History.
