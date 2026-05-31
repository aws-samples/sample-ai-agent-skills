---
title: "G1 — Experience Builder Issues"
description: "Diagnose Kendra Experience Builder creation and configuration"
status: active
severity: MEDIUM
triggers:
  - "experience creation failed"
  - "search application error"
  - "experience not loading"
owner: devops-agent
objective: "Resolve Experience Builder issues"
context: "Experience Builder creates search applications. Requires IAM Identity Center."
---

## Phase 1 — Triage

MUST:
- Check experience status: `aws kendra describe-experience --id <exp-id> --index-id <index-id>`
- Verify IAM Identity Center is configured
- Check experience endpoints
- Verify IAM role for experience

SHOULD:
- Check user/group assignments
- Verify content source configuration
- Test experience URL

MAY:
- Check for browser compatibility issues
- Review experience customization

## Phase 2 — Remediate

MUST:
- Configure IAM Identity Center if not set up
- Fix experience configuration
- Assign users/groups to experience

SHOULD:
- Test with different user roles
- Configure content sources

## Common Issues

- symptoms: "Experience creation fails with SSO error"
  diagnosis: "IAM Identity Center not configured in account."
  resolution: "Enable and configure IAM Identity Center. Then create experience."

## Output Format

```yaml
root_cause: "experience_builder — <specific_cause>"
evidence:
  - type: experience_status
    content: "<experience configuration>"
severity: MEDIUM
mitigation:
  immediate: "Fix experience configuration"
  long_term: "Document experience setup requirements"
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
