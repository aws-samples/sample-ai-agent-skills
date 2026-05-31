---
title: "B2 — Knowledge Base Sync Failures"
description: "Diagnose knowledge base data source sync failures"
status: active
severity: HIGH
triggers:
  - "sync failed"
  - "ingestion job error"
  - "data source sync"
  - "StartIngestionJob failed"
owner: devops-agent
objective: "Identify and resolve knowledge base sync failures"
context: "KB sync (ingestion) processes S3 documents, chunks text, generates embeddings, and stores in the vector database. Sync failures stem from S3 access issues, document format problems, embedding model errors, vector store write failures, or IAM permissions."
---

## Phase 1 — Triage

MUST:
- Check ingestion job status: `aws bedrock-agent get-ingestion-job --knowledge-base-id <kb-id> --data-source-id <ds-id> --ingestion-job-id <job-id>`
- List recent ingestion jobs: `aws bedrock-agent list-ingestion-jobs --knowledge-base-id <kb-id> --data-source-id <ds-id>`
- Verify S3 data source accessibility
- Check KB execution role permissions

SHOULD:
- Check for document format issues (supported: PDF, TXT, MD, HTML, DOC, CSV)
- Verify embedding model is accessible
- Check vector store write permissions
- Review ingestion job failure reasons

MAY:
- Check document sizes (max file size limits apply)
- Verify S3 bucket encryption compatibility

## Phase 2 — Remediate

MUST:
- Fix S3 access permissions for the KB role
- Ensure documents are in supported formats
- Verify vector store is accessible and writable

SHOULD:
- Start with a small dataset to test sync
- Monitor ingestion job progress
- Check for partial sync failures (some documents may fail)

MAY:
- Implement sync monitoring and alerting
- Create data validation pipelines before sync

## Common Issues

- symptoms: "Ingestion job fails with S3 access error"
  diagnosis: "KB role lacks S3 GetObject permission on the data source bucket."
  resolution: "Add s3:GetObject and s3:ListBucket permissions to the KB role."

- symptoms: "Sync completes but documents not indexed"
  diagnosis: "Document format not supported or parsing failed."
  resolution: "Verify document format. Check for corrupted files. Use supported formats."

## Output Format

```yaml
root_cause: "sync_failure — <specific_cause>"
evidence:
  - type: ingestion_status
    content: "<ingestion job status and errors>"
  - type: data_source
    content: "<S3 data source configuration>"
severity: HIGH
mitigation:
  immediate: "Fix sync blocker and retry ingestion"
  long_term: "Implement data validation and sync monitoring"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix S3 access permissions for the KB role | YELLOW |
| Ensure documents are in supported formats | GREEN |
| Verify vector store is accessible and writable | GREEN |
| Start with a small dataset to test sync | GREEN |
| Monitor ingestion job progress | GREEN |
| Check for partial sync failures | GREEN |
| Implement sync monitoring and alerting | GREEN |
| Create data validation pipelines before sync | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Ingestion jobs process documents that may contain sensitive or proprietary information

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If S3 permissions were broadened, revert the IAM policy or bucket policy to the previous version
2. If a sync ingested incorrect data, delete the knowledge base data source and re-sync with corrected data
3. If vector store data is corrupted, delete and recreate the vector index, then re-sync
4. Verify rollback by running `aws bedrock-agent list-ingestion-jobs` and confirming clean state

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
