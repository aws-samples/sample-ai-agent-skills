---
title: "D2 — SSM Automation Errors"
description: "Diagnose SSM Automation execution errors triggered by AWS Config remediation"
status: active
severity: HIGH
triggers:
  - "SSM automation failed"
  - "automation execution error"
  - "SSM document error"
  - "automation step failed"
  - "remediation SSM error"
owner: devops-agent
objective: "Identify and resolve SSM Automation execution failures that prevent Config remediation from completing"
context: "Config remediation triggers SSM Automation documents. SSM execution failures have their own error taxonomy — step failures, parameter validation errors, rate limiting, and assume-role issues. The SSM Automation execution details provide step-level diagnostics."
---

## Phase 1 — Triage

MUST:
- List recent automation executions: `aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed --max-results 20`
- Get execution details: `aws ssm get-automation-execution --automation-execution-id <execution-id>`
- Check the failed step output for error messages
- Verify the automation document: `aws ssm describe-document --name <document-name>`
- Check the assume role used by the automation: `aws iam get-role --role-name <automation-role>`

SHOULD:
- List automation step executions: `aws ssm describe-automation-step-executions --automation-execution-id <execution-id>`
- Check CloudTrail for the specific API calls made by the automation
- Verify the document parameters match expected types and values

MAY:
- Test the automation document manually with known-good parameters
- Check SSM service quotas: `aws service-quotas get-service-quota --service-code ssm --quota-code L-0568E572`

## Phase 2 — Remediate

MUST:
- Fix the identified step failure (permissions, parameters, resource state)
- Verify the automation assume role has all required permissions
- Re-run the automation after fixing the issue

SHOULD:
- Update the SSM document if the logic is incorrect
- Add error handling steps to the automation document
- Verify parameter mappings between Config remediation and SSM document

MAY:
- Create a custom SSM document with better error handling
- Set up CloudWatch Events for SSM Automation failure notifications

## Common Issues

- symptoms: "Automation step fails with 'AssumeRole is not authorized to perform'"
  diagnosis: "The automation assume role lacks the required permissions for the specific API call in the failed step."
  resolution: "Add the missing permissions to the automation assume role. Check the step output for the exact API call that failed."

- symptoms: "Automation fails with InvalidParameterValue"
  diagnosis: "The parameters passed from Config remediation do not match the SSM document's expected parameter types or values."
  resolution: "Review the remediation configuration parameter mapping and ensure values match the document's parameter definitions."

- symptoms: "Automation execution rate exceeded"
  diagnosis: "Too many concurrent automation executions are running, hitting the SSM Automation concurrency limit."
  resolution: "Reduce Config remediation concurrency settings or request an SSM Automation concurrency limit increase."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix SSM step failure | YELLOW | Fixes automation; reversible |
| Fix assume role permissions | YELLOW | Permission change; reversible |
| Re-run automation | YELLOW | Executes SSM automation; may change resources |

## Escalation Conditions

- Remediation action deployment
- Recorder configuration changes
- SSM Automation execution on production resources

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `get-automation-execution` | MEDIUM | Execution details |
| `describe-automation-step-executions` | MEDIUM | Step-level details |
| `describe-document` | LOW | SSM document configuration |

## Prohibited Actions

- NEVER suggest stopping Config recorder
- NEVER suggest deleting delivery channel
- NEVER suggest disabling organization rules
- NEVER deploy untested SSM documents to production

## Phase 3 — Rollback

| Remediation | Rollback Step |
|---|---|
| SSM step fix | Revert SSM document to previous version |
| Assume role fix | Revert IAM policy to previous version |
| Automation re-run | Revert resource changes made by automation |

## Output Format

```yaml
root_cause: "ssm_automation_error — <specific_cause>"
evidence:
  - type: automation_execution
    content: "<get-automation-execution output>"
  - type: step_executions
    content: "<failed step details>"
  - type: iam_role
    content: "<automation role permissions>"
severity: HIGH
mitigation:
  immediate: "Fix the step failure and re-run the automation"
  long_term: "Improve automation document error handling and monitoring"
```

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
  - "NEVER suggest disabling Config rules to fix compliance"
  - "NEVER suggest deleting the configuration recorder"
  - "NEVER remove remediation actions without review"
