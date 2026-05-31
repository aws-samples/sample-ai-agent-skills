---
title: "C2 — Retriever Configuration Issues"
description: "Diagnose retriever setup and configuration problems"
status: active
severity: MEDIUM
triggers:
  - "retriever error"
  - "retriever not working"
  - "native vs Kendra retriever"
owner: devops-agent
objective: "Resolve retriever configuration issues"
context: "Q Business supports native and Kendra retrievers. Configuration affects retrieval behavior."
---

## Phase 1 — Triage

MUST:
- List retrievers: `aws qbusiness list-retrievers --application-id <app-id>`
- Check retriever type and status
- Verify retriever configuration
- Check index association

SHOULD:
- Compare native vs Kendra retriever performance
- Check retriever-specific settings

## Phase 2 — Remediate

MUST:
- Fix retriever configuration
- Ensure index is properly associated
- Verify retriever status is ACTIVE

SHOULD:
- Test retrieval with both retriever types
- Document retriever choice rationale

## Common Issues

- symptoms: "Retriever shows FAILED status"
  diagnosis: "Configuration error or index issue."
  resolution: "Check retriever configuration. Verify associated index is healthy."

## Output Format

```yaml
root_cause: "retriever_config — <specific_cause>"
evidence:
  - type: retriever_status
    content: "<retriever configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix retriever configuration"
  long_term: "Monitor retriever health"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: MEDIUM — retriever configuration affects data access
- Service Disruption Risk: LOW — configuration issues affect retrieval quality
- Compliance Impact: MEDIUM — retriever must respect data boundaries

## Escalation Conditions
- Retriever configuration prevents data retrieval
- Configuration changes degrade retrieval quality
- Retriever index issues
- Configuration incompatible with data sources
- Multiple retriever configurations fail

## Data Sensitivity
- Retriever configurations — INTERNAL
- Index settings — INTERNAL
- Data source mappings — INTERNAL
- Performance metrics — INTERNAL
- Never expose retriever internals

## Prohibited Actions
- Do NOT modify retriever configurations without testing
- Do NOT bypass retriever validation
- Do NOT create retrievers without proper data access controls
- Do NOT ignore retriever performance degradation
- Do NOT share retriever configurations externally
- Do NOT disable retriever access controls

## Phase 3 — Rollback
- Restore previous retriever configurations
- Revert index changes
- Verify retrieval quality after rollback
- Restore data source mappings
- Document retriever issues
- Update retriever management procedures

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
