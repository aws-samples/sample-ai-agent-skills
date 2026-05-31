---
title: "D1 — Wisdom Knowledge Base Errors"
description: "Diagnose Amazon Connect Wisdom knowledge base creation and content issues"
status: active
severity: HIGH
triggers: ["knowledge base error", "cannot create knowledge base", "content not indexed", "Wisdom setup failed"]
owner: devops-agent
objective: "Resolve Wisdom knowledge base creation, content ingestion, and indexing issues"
context: "Wisdom knowledge bases store content that agents search during calls. Content sources include S3, ServiceNow, Salesforce, SharePoint, and Zendesk. Content must be indexed before it is searchable."
---

## Phase 1 — Triage

MUST:
- List knowledge bases: `aws wisdom list-knowledge-bases`
- Get knowledge base details: `aws wisdom get-knowledge-base --knowledge-base-id <kb-id>`
- Check knowledge base status (must be ACTIVE for search)

SHOULD:
- Check content sources: `aws wisdom list-contents --knowledge-base-id <kb-id>`
- Verify the content source connector is configured correctly
- Check if content indexing has completed

MAY:
- Review CloudTrail for Wisdom API errors
- Check IAM permissions for the Wisdom service role

## Phase 2 — Remediate

MUST:
- Create knowledge base if missing: `aws wisdom create-knowledge-base --name <name> --knowledge-base-type CUSTOM`
- Fix content source configuration (S3 bucket path, connector credentials)
- Trigger content sync if content is not indexed: `aws wisdom start-content-upload --knowledge-base-id <kb-id>`

SHOULD:
- Verify S3 bucket permissions allow Wisdom service access
- Ensure content is in supported formats (HTML, PDF, Word, plain text)
- Check connector credentials for third-party sources (ServiceNow, Salesforce)

MAY:
- Set up automatic content sync schedules
- Configure content metadata for improved search relevance
- Create separate knowledge bases for different content domains

## Safety Ratings

```
safety_ratings:
  - "list-knowledge-bases: GREEN — read-only enumeration"
  - "get-knowledge-base: GREEN — read-only details"
  - "list-contents: GREEN — read-only content listing"
  - "create-knowledge-base: YELLOW — creates knowledge base resource"
  - "start-content-upload: YELLOW — triggers content indexing"
```

## Escalation Conditions

- Knowledge base creation fails with internal service error
- Content indexing fails repeatedly for specific content types
- Third-party connector authentication fails after credential rotation
- Knowledge base status stuck in CREATE_IN_PROGRESS for more than 1 hour

## Data Sensitivity

- `aws wisdom list-knowledge-bases`: LOW — returns knowledge base metadata
- `aws wisdom get-knowledge-base`: LOW — returns knowledge base configuration
- `aws wisdom list-contents`: MEDIUM — reveals knowledge base content titles
- Knowledge base content: MEDIUM to HIGH — may contain proprietary business information

## Prohibited Actions

- NEVER delete a production knowledge base without backup
- NEVER upload customer PII as knowledge base content
- NEVER share third-party connector credentials in logs or tickets
- NEVER assume content is immediately searchable after upload — indexing takes time

## Phase 3 — Rollback

1. If a knowledge base was created in error, delete it: `aws wisdom delete-knowledge-base --knowledge-base-id <kb-id>`
2. If content was uploaded incorrectly, delete the content: `aws wisdom delete-content --knowledge-base-id <kb-id> --content-id <content-id>`
3. Restore previous connector configuration if changes broke content sync
4. Verify agent search functionality after any changes
5. Document the knowledge base change and rollback reason

## Common Issues

- symptoms: "Knowledge base is ACTIVE but search returns no results"
  diagnosis: "Content has been uploaded but not yet indexed — indexing is asynchronous."
  resolution: "Wait for indexing to complete. Check content status for indexing progress."

## Output Format

```yaml
root_cause: "wisdom_knowledge_base — <specific_cause>"
evidence:
  - type: kb_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix knowledge base configuration"
  long_term: "Implement content management and sync monitoring"
```

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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
