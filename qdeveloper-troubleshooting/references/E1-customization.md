---
title: "E1 — Customization Issues"
description: "Diagnose custom model creation and deployment"
status: active
severity: MEDIUM
triggers:
  - "customization failed"
  - "custom model error"
  - "training failed"
owner: devops-agent
objective: "Resolve customization issues"
context: "Customization trains models on your codebase. Requires Pro tier."
---

## Phase 1 — Triage

MUST:
- Verify Pro tier subscription
- Check customization job status
- Verify training data in S3
- Check IAM permissions

SHOULD:
- Review training data quality
- Check for data format issues
- Verify S3 bucket access

## Phase 2 — Remediate

MUST:
- Fix training data issues
- Ensure Pro tier is active
- Fix IAM permissions

SHOULD:
- Validate training data before submission
- Monitor customization job progress

## Common Issues

- symptoms: "Customization job fails with data error"
  diagnosis: "Training data format or quality issues."
  resolution: "Review training data requirements. Fix format. Resubmit."

## Output Format

```yaml
root_cause: "customization — <specific_cause>"
evidence:
  - type: customization_status
    content: "<customization job details>"
severity: MEDIUM
mitigation:
  immediate: "Fix training data"
  long_term: "Automate data validation"
```


## Safety Ratings
- Overall Risk: MEDIUM
- Data Exposure Risk: HIGH — customization involves proprietary code repositories
- Service Disruption Risk: LOW — customization issues affect suggestion quality
- Compliance Impact: HIGH — customization must comply with code usage policies

## Escalation Conditions
- Customization creation fails
- Customization degrades suggestion quality
- Customization accesses unauthorized repositories
- Customization training fails
- Customization costs exceed budget

## Data Sensitivity
- Code repositories used for customization — CONFIDENTIAL
- Customization configurations — CONFIDENTIAL
- Training metrics — INTERNAL
- Repository access details — CONFIDENTIAL
- Never expose customization training data

## Prohibited Actions
- Do NOT customize with unauthorized code repositories
- Do NOT bypass customization governance review
- Do NOT share customization configurations externally
- Do NOT customize with code containing known vulnerabilities
- Do NOT ignore customization quality degradation
- Do NOT train on code with incompatible licenses

## Phase 3 — Rollback
- Delete failed customizations
- Restore previous customization configurations
- Revert repository access changes
- Verify suggestion quality after rollback
- Document customization issues
- Update customization procedures

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
