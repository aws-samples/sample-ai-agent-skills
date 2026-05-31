# AWS Data Exchange Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any Data Exchange issue.

## Guardrail 1: Provider vs Subscriber Roles Are Distinct
Providers publish data sets and manage revisions. Subscribers discover, subscribe to, and consume data. Each role has different IAM permissions, API access, and workflows. Do not confuse provider-side operations with subscriber-side operations.

## Guardrail 2: Revisions Must Be Finalized
Revisions are created as drafts and must be explicitly finalized before they are visible to subscribers. Unfinalized revisions exist only in the provider's account. Do not assume a created revision is immediately available to subscribers.

## Guardrail 3: Entitlement Does Not Mean Data Is Copied
When a subscriber is entitled to a data set, the data remains in the provider's account. Subscribers must explicitly export data to their own S3 bucket, query via Redshift, or access via Lake Formation. Entitlement grants access rights, not data copies.

## Guardrail 4: Export Jobs Are Asynchronous
S3 export jobs run asynchronously. Creating an export job returns a job ID that must be polled for completion. Jobs can take minutes to hours depending on data size. Do not assume data is available immediately after creating an export job.

## Guardrail 5: API Assets Proxy to Provider Backend
API assets in Data Exchange proxy HTTP requests to the provider's API Gateway endpoint. Errors (4xx, 5xx) may originate from the provider's backend, not from Data Exchange itself. Distinguish between Data Exchange service errors and provider API errors.

## Guardrail 6: Lake Formation and Redshift Use AWS RAM
Lake Formation data sets and Redshift data shares use AWS Resource Access Manager (RAM) for cross-account sharing. RAM resource share invitations must be accepted before data is accessible. Check RAM invitation status before diagnosing access issues.

## Guardrail 7: Data Exchange Is Regional
Data Exchange operates in specific AWS regions. Data sets, subscriptions, and entitlements are regional. A subscription in us-east-1 does not grant access to the same data set in eu-west-1. Verify the correct region for all operations.

## Guardrail 8: Marketplace Integration Has Separate Billing
Data Exchange products listed on AWS Marketplace have separate billing through AWS Marketplace. Subscription costs, metering, and billing issues are handled by Marketplace, not Data Exchange directly. Check Marketplace billing for cost-related issues.

## Guardrail 9: Event Actions Require S3 Event Configuration
Automatic revision publishing via event actions requires correct S3 event notification configuration. The S3 bucket must send events to Data Exchange. Missing or misconfigured S3 events prevent automatic publishing.

## Guardrail 10: Asset Types Determine Access Methods
Data Exchange supports multiple asset types: S3 snapshots, API Gateway APIs, Redshift data shares, and Lake Formation data permissions. Each asset type has different access methods and requirements. Do not apply S3 export procedures to Redshift or Lake Formation assets.

## Guardrail 11: Subscription Verification Requires Provider Approval
Some data products require provider approval before subscription is activated. Pending subscriptions are not yet entitled. Check subscription status (ACTIVE vs PENDING) before diagnosing access issues.

## Guardrail 12: Data Set Deletion Is Irreversible
Deleting a data set removes all revisions and assets permanently. Active subscriptions are terminated. There is no undelete capability. Verify there are no active subscribers before deleting a data set.
