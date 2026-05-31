---
name: workdocs-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Amazon WorkDocs
  problems by analyzing site creation, directory integration, upload/download
  failures, sync client issues, sharing permissions, collaboration,
  user management, admin controls, migration, storage, and API integration.
  Activate when: site creation failures, directory integration issues,
  upload/download errors, sync client problems, sharing permission errors,
  user management issues, migration failures, storage limits, or the user
  says something is wrong with WorkDocs.
compatibility: >
  Requires AWS CLI or SDK access with workdocs, ds (Directory Service),
  iam, s3, cloudtrail, and cloudwatch permissions.
---

# Amazon WorkDocs Diagnostics

## When to use

Any Amazon WorkDocs investigation — site creation, directory integration, upload/download failures, sync client issues, sharing permissions, collaboration problems, user management, admin controls, migration, storage limits, or API integration.

## Investigation workflow

### Step 1 — Collect and triage

```
aws workdocs describe-users --organization-id <org-id> --query 'Users[*].{Username:Username,Status:Status,Type:Type}'
aws workdocs describe-folder-contents --folder-id <folder-id> --type ALL
aws workdocs describe-activities --organization-id <org-id> --limit 20
```

### Step 2 — Domain deep dive

```
aws workdocs get-document --document-id <doc-id>
aws workdocs get-document-version --document-id <doc-id> --version-id <version-id>
aws workdocs describe-resource-permissions --resource-id <resource-id>
```

### Step 3 — Detailed investigation

```
aws ds describe-directories --query 'DirectoryDescriptions[*].{Id:DirectoryId,Name:Name,Type:Type,Status:Stage}'
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=workdocs.amazonaws.com --max-results 20
aws workdocs describe-notification-subscriptions --organization-id <org-id>
```

Read `references/guardrails.md` before concluding on any WorkDocs issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `workdocs describe-users` | List users and their status |
| `workdocs get-document` | Get document metadata |
| `workdocs describe-folder-contents` | List folder contents |
| `workdocs describe-resource-permissions` | Check sharing permissions |
| `workdocs describe-activities` | View activity feed |
| `workdocs create-user` | Create a new user |
| `ds describe-directories` | Check directory integration |

## Gotchas: Amazon WorkDocs

- WorkDocs requires an AWS Directory Service directory (Simple AD, AD Connector, or AWS Managed Microsoft AD). No directory means no WorkDocs site.
- WorkDocs sites are REGIONAL. Each site is tied to a specific region and directory. Users and content do not replicate across regions.
- User types matter: ADMIN, USER, POWERUSER, and MINIMALUSER have different capabilities and storage quotas. Changing user type affects permissions.
- File size limits exist: maximum single file upload is 5 GB via the API. The web client and sync client may have lower practical limits.
- Sharing permissions are hierarchical: folder permissions cascade to contained documents unless explicitly overridden.
- The WorkDocs sync client is a separate desktop application. Sync issues are often client-side (network, local disk, OS permissions) not service-side.
- WorkDocs API requires authentication tokens for user-level operations. Admin operations use IAM credentials.

## Anti-hallucination rules

1. Always cite specific organization IDs, user IDs, or API responses as evidence.
2. WorkDocs is NOT S3. Never suggest S3 APIs for WorkDocs content management.
3. Directory Service integration is required. Never suggest WorkDocs without a directory.
4. User types have different capabilities. Never assume all users have the same permissions.
5. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 12 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Sites | A1–A2 | Site creation failures, directory integration |
| B — Content | B1–B2 | Upload/download failures, sync client issues |
| C — Sharing | C1–C2 | Sharing permission errors, collaboration issues |
| D — Users | D1–D2 | User management, admin controls |
| E — Operations | E1–E2 | Migration issues, storage limits |
| F — Integration | F1 | API integration |
| Z — Catch-All | Z1 | General troubleshooting |
