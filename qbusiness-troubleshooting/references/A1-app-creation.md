---
title: "A1 — Application Creation Failures"
description: "Diagnose Q Business application creation issues"
status: active
severity: HIGH
triggers:
  - "application creation failed"
  - "cannot create Q Business app"
  - "CreateApplication error"
owner: devops-agent
objective: "Resolve Q Business application creation failures"
context: "Application creation requires IAM Identity Center, proper IAM role, and service quotas."
---

## Phase 1 — Triage

MUST:
- Verify IAM Identity Center is enabled: `aws sso-admin list-instances`
- Check IAM permissions for qbusiness:CreateApplication
- Verify service quotas for Q Business applications
- Review CloudTrail for creation errors

SHOULD:
- Check Identity Center instance ARN
- Verify encryption configuration (KMS key if specified)
- Check VPC configuration if using VPC mode

MAY:
- Check for regional availability
- Review account-level limits

## Phase 2 — Remediate

MUST:
- Enable IAM Identity Center if not configured
- Fix IAM permissions
- Request quota increase if at limit

SHOULD:
- Document application configuration
- Use IaC for reproducible setup

## Common Issues

- symptoms: "ValidationException: Identity Center instance not found"
  diagnosis: "IAM Identity Center not enabled in account/region."
  resolution: "Enable IAM Identity Center in the same region as Q Business."

## Output Format

```yaml
root_cause: "app_creation — <specific_cause>"
evidence:
  - type: app_status
    content: "<creation error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation prerequisites"
  long_term: "Template application configurations"
```


## Safety Ratings
- Overall Risk: HIGH
- Data Exposure Risk: HIGH — Q Business apps access enterprise data sources
- Service Disruption Risk: LOW — creation failures do not affect existing apps
- Compliance Impact: HIGH — app configuration affects data access and privacy

## Escalation Conditions
- App creation fails with IAM or Identity Center errors
- App creation blocked by organizational policies
- Encryption configuration fails during creation
- App creation leaves resources in inconsistent state
- Multiple app creations fail simultaneously

## Data Sensitivity
- Identity Center configurations — CONFIDENTIAL
- IAM role configurations — CONFIDENTIAL
- Encryption key settings — CONFIDENTIAL
- App configurations — INTERNAL
- Never expose Identity Center credentials or app admin tokens

## Prohibited Actions
- Do NOT create apps without proper encryption configuration
- Do NOT use overly permissive IAM roles
- Do NOT skip Identity Center integration for production apps
- Do NOT create apps without data access controls
- Do NOT share app admin credentials
- Do NOT bypass data governance review for new apps

## Phase 3 — Rollback
- Delete failed app resources
- Clean up orphaned IAM roles
- Revert Identity Center configuration changes
- Document creation failure reasons before retrying
- Verify no residual resources remain
- Update app creation procedures

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
