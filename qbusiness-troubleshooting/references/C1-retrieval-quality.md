---
title: "C1 — Retrieval Quality Issues"
description: "Diagnose poor answer quality and missing results"
status: active
severity: MEDIUM
triggers:
  - "poor answers"
  - "wrong results"
  - "missing information"
owner: devops-agent
objective: "Improve retrieval quality"
context: "Retrieval quality depends on document quality, chunking, and index configuration."
---

## Phase 1 — Triage

MUST:
- Test chat with known queries
- Verify documents are synced
- Check if ACLs are filtering results
- Review retriever configuration

SHOULD:
- Check document chunking settings
- Verify metadata is indexed
- Test with different query formulations

## Phase 2 — Remediate

MUST:
- Ensure documents are synced
- Fix ACL issues if filtering results
- Adjust chunking strategy

SHOULD:
- Add metadata to documents
- Test with representative queries

## Common Issues

- symptoms: "Chat returns 'I don't have information about that'"
  diagnosis: "Documents not synced or ACLs blocking access."
  resolution: "Verify sync completed. Check user ACL access."

## Output Format

```yaml
root_cause: "retrieval_quality — <specific_cause>"
evidence:
  - type: chat_response
    content: "<query and response details>"
severity: MEDIUM
mitigation:
  immediate: "Fix retrieval blockers"
  long_term: "Optimize chunking and metadata"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — retrieval accesses enterprise knowledge base
- Service Disruption Risk: LOW — quality issues affect relevance, not availability
- Compliance Impact: HIGH — retrieval must respect data access controls

## Escalation Conditions
- Retrieval returns irrelevant or incorrect results
- Retrieval bypasses access controls
- Retrieval quality degrades significantly
- Retrieval returns sensitive data to unauthorized users
- Retrieval performance exceeds latency thresholds

## Data Sensitivity
- Retrieved documents — CONFIDENTIAL
- Query patterns — INTERNAL
- Retrieval metrics — INTERNAL
- User access patterns — CONFIDENTIAL
- Never expose retrieved content outside authorized context

## Prohibited Actions
- Do NOT bypass retrieval access controls
- Do NOT ignore retrieval quality degradation
- Do NOT expose retrieved content to unauthorized users
- Do NOT modify retrieval configurations without testing
- Do NOT disable retrieval access controls
- Do NOT share retrieval results externally

## Phase 3 — Rollback
- Restore previous retrieval configurations
- Revert retriever settings
- Verify retrieval quality after changes
- Restore access control configurations
- Document retrieval issues and quality metrics
- Update retrieval optimization procedures

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
