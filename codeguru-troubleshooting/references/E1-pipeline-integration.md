---
title: "E1 — CI/CD Pipeline Integration"
description: "Diagnose CodeGuru integration with CI/CD pipelines"
status: active
severity: MEDIUM
triggers:
  - "pipeline integration error"
  - "CodeGuru step failed"
  - "automated review not running"
owner: devops-agent
objective: "Resolve CI/CD integration issues"
context: "CodeGuru can be integrated into CI/CD pipelines for automated reviews."
---

## Phase 1 — Triage

MUST:
- Check pipeline step configuration
- Verify IAM role for pipeline
- Check CodeGuru API calls in pipeline logs
- Verify repository association

SHOULD:
- Check pipeline timeout settings
- Verify S3 access for code artifacts

## Phase 2 — Remediate

MUST:
- Fix pipeline IAM permissions
- Correct API call parameters
- Ensure association exists

SHOULD:
- Add error handling to pipeline
- Configure review result gates

## Common Issues

- symptoms: "Pipeline step fails with AccessDenied"
  diagnosis: "Pipeline IAM role missing CodeGuru permissions."
  resolution: "Add codeguru-reviewer:CreateCodeReview to pipeline role."

## Output Format

```yaml
root_cause: "pipeline_integration — <specific_cause>"
evidence:
  - type: pipeline_logs
    content: "<pipeline execution details>"
severity: MEDIUM
mitigation:
  immediate: "Fix pipeline configuration"
  long_term: "Automate review gates"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Fix pipeline IAM permissions | YELLOW |
| Correct API call parameters | GREEN |
| Ensure association exists | GREEN |
| Add error handling to pipeline | GREEN |
| Configure review result gates | YELLOW |

## Escalation Conditions

- Fix requires modifying production CI/CD pipeline configurations
- Fix involves changing pipeline IAM roles

## Data Sensitivity

- **MEDIUM**: pipeline integration configurations expose CI/CD architecture and review workflows
- Pipeline logs may contain code review results and security findings

## Prohibited Actions

- NEVER suggest suppressing all security findings

## Phase 3 — Rollback

1. If pipeline IAM permissions were modified, revert to previous policy
2. If pipeline step configuration was changed, restore previous pipeline definition
3. If review result gates were added, remove if blocking deployments incorrectly
4. Verify rollback by confirming pipeline executes successfully

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
