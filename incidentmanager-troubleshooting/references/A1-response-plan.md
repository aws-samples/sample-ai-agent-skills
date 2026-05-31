---
title: "A1 — Response Plan Issues"
description: "Diagnose Incident Manager response plan creation and configuration problems"
status: active
severity: HIGH
triggers:
  - "response plan"
  - "create response plan"
  - "response plan error"
  - "response plan configuration"
owner: devops-agent
objective: "Resolve response plan creation and configuration issues"
context: "Response plans define how incidents are handled — which contacts to engage, what runbooks to execute, and what metrics to track. A replication set must exist before creating response plans. Common issues include missing replication set, invalid SSM document references, and permission errors."
---

## Phase 1 — Triage

MUST:
- Verify replication set exists: `aws ssm-incidents get-replication-set`
- List existing response plans: `aws ssm-incidents list-response-plans`
- If specific plan, get details: `aws ssm-incidents get-response-plan --arn <plan-arn>`
- Verify IAM permissions include `ssm-incidents:CreateResponsePlan`
- If referencing SSM document, verify it exists: `aws ssm describe-document --name <doc-name>`

SHOULD:
- Check CloudTrail for response plan creation errors
- Verify contacts and escalation plans referenced in the plan exist
- Check that the execution role for automation has required permissions

MAY:
- Validate response plan JSON structure
- Check service quotas for response plans

## Phase 2 — Remediate

MUST:
- If no replication set, create one: `aws ssm-incidents create-replication-set --regions '{"<region>":{"sseKmsKeyId":"DefaultKey"}}'`
- If SSM document missing, create it before the response plan
- Fix IAM permissions for response plan creation

SHOULD:
- Test response plan with a simulated incident
- Configure engagements for each impact level

## Output Format

```yaml
root_cause: "response_plan — <specific_cause>"
evidence:
  - type: replication_set
    content: "<replication set status>"
  - type: plan_config
    content: "<response plan details>"
severity: HIGH
mitigation:
  immediate: "Fix prerequisites and create/update response plan"
  long_term: "Manage response plans via IaC and test regularly"
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
