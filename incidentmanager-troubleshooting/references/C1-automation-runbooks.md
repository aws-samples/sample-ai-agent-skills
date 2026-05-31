---
title: "C1 — Automation Runbook Issues"
description: "Diagnose Incident Manager automation runbook failures including SSM document and permission problems"
status: active
severity: HIGH
triggers:
  - "automation runbook"
  - "runbook failed"
  - "ssm automation incident"
  - "runbook not executing"
  - "automation error"
owner: devops-agent
objective: "Resolve automation runbook failures during incident response"
context: "Incident Manager uses SSM Automation documents as runbooks. They execute automatically when an incident is created. Common issues include missing SSM documents, execution role permissions, parameter mismatches, and document version conflicts."
---

## Phase 1 — Triage

MUST:
- Check response plan automation config: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- Verify SSM document exists: `aws ssm describe-document --name <doc-name>`
- Check execution role permissions: `aws iam get-role --role-name <role-name>`
- List automation executions: `aws ssm describe-automation-executions --filters Key=DocumentNamePrefix,Values=<doc-name>`
- Check execution status and failure details

SHOULD:
- Verify document parameters match response plan configuration
- Check execution role trust policy allows SSM to assume it
- Review automation execution output for error details

MAY:
- Test automation manually: `aws ssm start-automation-execution --document-name <doc-name> --parameters <params>`
- Check CloudTrail for automation execution events

## Phase 2 — Remediate

MUST:
- If document missing, create the SSM Automation document
- If role permissions insufficient, add required permissions
- If parameter mismatch, update response plan or document parameters
- Ensure execution role trust policy includes `ssm.amazonaws.com`

SHOULD:
- Test runbooks independently before associating with response plans
- Use document versions to prevent breaking changes

## Output Format

```yaml
root_cause: "automation_runbook — <specific_cause>"
evidence:
  - type: automation_config
    content: "<response plan automation configuration>"
  - type: execution_status
    content: "<automation execution details>"
severity: HIGH
mitigation:
  immediate: "Fix SSM document, permissions, or parameters"
  long_term: "Test runbooks regularly and version SSM documents"
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
- Never delete response plans that are actively referenced by CloudWatch alarms
- Never modify escalation plans during active incidents
- Never remove contact channels without confirming alternative notification paths exist
- Never delete incident records or post-incident analysis data
- Never disable automation runbooks tied to active response plans without backup procedures

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous response plan configuration if incident detection breaks
- Revert escalation plan changes if notifications fail to reach responders
- Re-enable previous contact channel settings if engagement plans stop working

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
