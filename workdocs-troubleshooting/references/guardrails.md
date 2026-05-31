# Amazon WorkDocs Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any WorkDocs issue.

## Guardrail 1: Directory Service Is Required
WorkDocs requires an AWS Directory Service directory (Simple AD, AD Connector, or AWS Managed Microsoft AD). Without a directory, a WorkDocs site cannot be created. Directory health directly affects WorkDocs availability.

## Guardrail 2: Sites Are Regional
Each WorkDocs site is tied to a specific AWS region and directory. Users, documents, and folders do not replicate across regions. If users report missing content, verify they are accessing the correct regional site.

## Guardrail 3: User Types Have Different Capabilities
WorkDocs has four user types: ADMIN, POWERUSER, USER, and MINIMALUSER. Each has different storage quotas and feature access. MINIMALUSER can only view shared content. Do not assume all users have the same capabilities.

## Guardrail 4: Authentication Tokens vs IAM Credentials
WorkDocs API operations use two authentication methods. Admin operations (describe-users, describe-activities) use IAM credentials. User-level operations (get-document, create-folder) require an authentication token obtained via CreateNotificationSubscription or the console.

## Guardrail 5: File Size Limits Apply
Maximum single file upload via API is 5 GB. The web client and sync client may have lower practical limits depending on browser and network conditions. Large file uploads may time out without indicating a service error.

## Guardrail 6: Sharing Permissions Are Hierarchical
Folder permissions cascade to all contained documents and subfolders unless explicitly overridden at the resource level. Removing folder-level sharing does not automatically remove explicit document-level shares.

## Guardrail 7: Sync Client Is a Separate Application
The WorkDocs sync client is a desktop application that runs locally. Sync issues are often caused by local factors: network connectivity, disk space, OS permissions, antivirus software, or client version. Do not assume sync failures are service-side.

## Guardrail 8: Deleted Content Has Recovery Window
Deleted documents go to the user's Recycle Bin and can be recovered within the retention period (60 days by default). Permanently deleted content cannot be recovered. Do not tell users content is permanently lost without checking the Recycle Bin.

## Guardrail 9: Migration Requires Careful Planning
Migrating content to WorkDocs from other systems requires the Amazon WorkDocs Migration Service or API-based migration. Large migrations should be planned with bandwidth and API rate limits in mind. There is no one-click migration.

## Guardrail 10: Storage Quotas Are Per-User
Storage quotas are assigned per user based on user type. Organization-level storage is the sum of all user quotas. Exceeding a user's quota blocks that user's uploads but does not affect other users.

## Guardrail 11: Activity Feed Has Retention Limits
The activity feed (describe-activities) retains events for a limited period. Historical activities beyond the retention window are not available via the API. For long-term audit, use CloudTrail.

## Guardrail 12: WorkDocs Is Not a Backup Service
WorkDocs is a document collaboration service, not a backup or archive solution. It does not provide point-in-time recovery, cross-region replication, or lifecycle policies. Do not recommend it as a replacement for S3 or Glacier for backup purposes.
