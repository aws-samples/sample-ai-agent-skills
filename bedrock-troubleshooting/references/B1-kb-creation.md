---
title: "B1 — Knowledge Base Creation Issues"
description: "Diagnose knowledge base creation failures in Bedrock"
status: active
severity: HIGH
triggers:
  - "knowledge base creation failed"
  - "KB creation error"
  - "vector store configuration"
  - "CreateKnowledgeBase error"
owner: devops-agent
objective: "Identify and resolve knowledge base creation failures"
context: "Knowledge base creation requires a pre-configured vector store, embedding model access, IAM role with proper permissions, and S3 data source. Common failures: vector store misconfiguration, wrong embedding dimensions, IAM role issues, or S3 access problems."
---

## Phase 1 — Triage

MUST:
- Check KB status: `aws bedrock-agent get-knowledge-base --knowledge-base-id <id>`
- Verify vector store is created and accessible (OpenSearch Serverless, Pinecone, etc.)
- Check embedding model access: verify model access for the embedding model
- Verify KB execution role permissions

SHOULD:
- Check vector index dimensions match embedding model (e.g., 1024 for Titan V2)
- Verify S3 data source bucket exists and is accessible
- Check CloudTrail for CreateKnowledgeBase events
- Verify OpenSearch Serverless collection is ACTIVE (if using AOSS)

MAY:
- Check for service quotas on knowledge bases
- Verify network configuration for vector store access

## Phase 2 — Remediate

MUST:
- Create and configure vector store before KB creation
- Match vector index dimensions to embedding model
- Create IAM role with bedrock, s3, and vector store permissions

SHOULD:
- Use OpenSearch Serverless for managed vector store
- Test vector store connectivity independently
- Verify embedding model is accessible in the region

MAY:
- Create KB creation automation with pre-validation
- Set up monitoring for KB health

## Common Issues

- symptoms: "KB creation fails with vector store error"
  diagnosis: "Vector store not created, wrong dimensions, or inaccessible."
  resolution: "Create vector store with correct dimensions. Verify accessibility."

- symptoms: "KB creation fails with role error"
  diagnosis: "KB execution role lacks required permissions."
  resolution: "Add bedrock:InvokeModel, s3:GetObject, and vector store permissions to the role."

## Output Format

```yaml
root_cause: "kb_creation — <specific_cause>"
evidence:
  - type: kb_status
    content: "<knowledge base status>"
  - type: vector_store
    content: "<vector store configuration>"
severity: HIGH
mitigation:
  immediate: "Fix vector store or role configuration"
  long_term: "Create KB creation templates with pre-validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Create and configure vector store before KB creation | YELLOW |
| Match vector index dimensions to embedding model | GREEN |
| Create IAM role with bedrock, s3, and vector store permissions | YELLOW |
| Use OpenSearch Serverless for managed vector store | YELLOW |
| Test vector store connectivity independently | GREEN |
| Verify embedding model is accessible in the region | GREEN |
| Create KB creation automation with pre-validation | GREEN |
| Set up monitoring for KB health | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Knowledge base data sources may contain proprietary or sensitive documents

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If a knowledge base was created incorrectly, delete it: `aws bedrock-agent delete-knowledge-base --knowledge-base-id <id>`
2. If a vector store was provisioned, delete the OpenSearch Serverless collection or vector index
3. If IAM roles were created, delete them after detaching all policies
4. If S3 data source permissions were broadened, revert the bucket policy to the previous version
5. Verify rollback by confirming `aws bedrock-agent list-knowledge-bases` no longer shows the erroneous KB

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
