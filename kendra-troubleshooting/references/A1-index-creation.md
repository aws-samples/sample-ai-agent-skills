---
title: "A1 — Index Creation Failures"
description: "Diagnose Kendra index creation and provisioning issues"
status: active
severity: HIGH
triggers:
  - "index creation failed"
  - "index stuck in CREATING"
  - "cannot create index"
owner: devops-agent
objective: "Resolve Kendra index creation failures"
context: "Index creation takes 15-30 minutes. Failures stem from IAM, quotas, or configuration errors."
---

## Phase 1 — Triage

MUST:
- Check index status: `aws kendra describe-index --id <index-id>`
- Verify IAM role has kendra:* and required service permissions
- Check service quotas for max indices per account
- Review CloudTrail for CreateIndex errors

SHOULD:
- Verify edition selection (Developer vs Enterprise)
- Check VPC configuration if using VPC mode
- Verify KMS key permissions if using CMK encryption

MAY:
- Check for regional availability issues
- Review account-level Kendra limits

## Phase 2 — Remediate

MUST:
- Fix IAM role permissions
- Request quota increase if at limit
- Recreate index with correct configuration

SHOULD:
- Use Developer Edition for testing
- Document index configuration for reproducibility

## Common Issues

- symptoms: "Index stuck in CREATING for over 1 hour"
  diagnosis: "VPC configuration issue or IAM permission problem."
  resolution: "Check VPC subnet/security group config. Verify IAM role trust policy."

## Output Format

```yaml
root_cause: "index_creation — <specific_cause>"
evidence:
  - type: index_status
    content: "<index creation status>"
severity: HIGH
mitigation:
  immediate: "Fix creation blockers"
  long_term: "Template index configurations"
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete indexes that serve production search applications
- Never modify data source connectors during active sync operations
- Never remove FAQ entries without confirming impact on search quality
- Never alter access control configurations without security review
- Never delete experience builder applications used by end users

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous data source connector configuration if sync operations fail
- Revert index relevance tuning settings if search quality degrades
- Re-enable previous ACL configurations if access control breaks

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
