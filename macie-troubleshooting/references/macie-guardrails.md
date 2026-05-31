# Amazon Macie Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Amazon Macie issue.

## Guardrail 1: Macie Scans S3 Only
Macie is designed exclusively for Amazon S3 data classification and sensitive data discovery. It does not scan RDS databases, DynamoDB tables, EBS volumes, EFS file systems, or any other data store. Do not suggest Macie for non-S3 sensitive data detection. Use other tools (e.g., database activity monitoring) for non-S3 data stores.

## Guardrail 2: Discovery Jobs Are Not Real-Time
Discovery jobs run on a schedule (daily, weekly, monthly) or as one-time jobs. Objects added to S3 after a job starts are not scanned until the next job run. For near-real-time detection, use scheduled jobs with daily frequency. Automated sensitive data discovery (if enabled) provides ongoing analysis but is separate from discovery jobs.

## Guardrail 3: Custom Data Identifiers Use Regex with Size Limits
Custom data identifiers use regular expressions (regex) to match patterns. Maximum regex length is 512 characters. Complex regex can cause timeouts. Regex must be compatible with PCRE (Perl Compatible Regular Expressions). Test patterns with sample data before deploying. Keywords and proximity rules can refine matches.

## Guardrail 4: Managed Identifiers Detect Common PII Types
Macie includes managed data identifiers for common sensitive data types: SSN, credit card numbers, passport numbers, email addresses, AWS credentials, etc. These cannot be modified. Custom identifiers supplement managed ones. Both contribute to findings. You can enable/disable specific managed identifiers per job.

## Guardrail 5: Findings Are Regional
Macie findings exist in the region where the S3 bucket is located. Cross-region visibility requires enabling Macie in each region and using Security Hub aggregation. The Macie console shows only findings for the current region. Do not assume findings from one region appear in another.

## Guardrail 6: Bucket Inventory Does Not Mean Buckets Are Scanned
Macie maintains an inventory of all S3 buckets in the account. This inventory shows bucket metadata (size, encryption, public access). Having a bucket in inventory does NOT mean it is being scanned for sensitive data. Discovery jobs must be created to scan specific buckets.

## Guardrail 7: Sampling Reduces Cost but May Miss Sensitive Data
Sampling depth controls what percentage of objects in a bucket are scanned. Lower sampling reduces cost but increases the risk of missing sensitive data. Use 100% sampling for compliance-critical buckets. Sampling is configured per discovery job. Default sampling depth varies by job type.

## Guardrail 8: Macie Must Be Enabled Per Region
Macie is a regional service. It must be enabled separately in each region where you want to scan S3 buckets. Enabling Macie in us-east-1 does not enable it in eu-west-1. Multi-region deployment requires per-region configuration.

## Guardrail 9: Suppression Rules Hide Findings, Not Fix Issues
Suppression rules filter findings from the active view. The sensitive data still exists in S3. Suppressed findings still count toward quotas. Use suppression for accepted risks after proper assessment. Always prefer data remediation (encryption, deletion, access restriction) over suppression.

## Guardrail 10: Pricing Is Per Object Evaluated
Macie charges per S3 object evaluated by discovery jobs. Scanning large buckets with millions of objects can be expensive. Use bucket-level cost estimates before running jobs. Automated sensitive data discovery has separate pricing. Check get-usage-totals for current costs.

## Guardrail 11: Classification Results Include Severity and Confidence
Findings include severity (Low, Medium, High) based on the type and volume of sensitive data found. Confidence scores indicate how likely the detection is accurate. Low confidence findings may be false positives. Review confidence scores before taking action.

## Guardrail 12: Allow Lists Reduce False Positives
Allow lists define text or patterns that should NOT be flagged as sensitive data. Use allow lists for known safe values (test data, public information). Allow lists can be S3 objects (text lists) or regex patterns. They apply to all discovery jobs in the account/region.
