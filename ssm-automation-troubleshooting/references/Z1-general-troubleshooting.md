---
title: "Z1 — General Troubleshooting"
description: "General SSM Automation troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["ssm automation issue", "automation not working", "general automation problem", "ssm automation help"]
owner: devops-agent
objective: "Provide general troubleshooting guidance for SSM Automation issues"
context: "This catch-all runbook covers general SSM Automation troubleshooting when the issue does not clearly map to a specific runbook. Start here and escalate to specific runbooks as the issue becomes clearer."
---

## Phase 1 — Triage

MUST:
- List recent automation executions: `aws ssm describe-automation-executions --max-results 20`
- Check for failed executions: `aws ssm describe-automation-executions --filters Key=ExecutionStatus,Values=Failed --max-results 10`
- Verify SSM agent connectivity if targeting instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{Id:InstanceId,Ping:PingStatus,Agent:AgentVersion}'`

SHOULD:
- Check service quotas: `aws service-quotas list-service-quotas --service-code ssm --query 'Quotas[?contains(QuotaName,`automation`)]'`
- Review CloudTrail for SSM events: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20`
- Check CloudWatch metrics for SSM Automation

MAY:
- Review AWS Health Dashboard for SSM service issues
- Check if the issue is region-specific by testing in another region
- Review recent IAM policy changes that may affect SSM permissions

## Phase 2 — Remediate

MUST:
- Address the specific error identified during triage
- Refer to the appropriate specific runbook (A1-G1) based on findings
- Verify the fix resolves the issue

SHOULD:
- Document the issue and resolution for future reference
- Set up monitoring for the specific failure mode
- Review related automation configurations for similar issues

MAY:
- Implement preventive measures (pre-validation, monitoring, alerts)
- Create custom CloudWatch dashboards for automation monitoring
- Review and update automation documents based on lessons learned

## Safety Ratings

```
safety_ratings:
  - "describe-automation-executions: GREEN — read-only execution listing"
  - "describe-instance-information: GREEN — read-only instance status"
  - "service-quotas list-service-quotas: GREEN — read-only quota information"
  - "cloudtrail lookup-events: GREEN — read-only audit log access"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook after initial triage
- Multiple automation features are failing simultaneously
- SSM service appears to be experiencing regional issues
- Issue requires AWS Support engagement

## Data Sensitivity

- `aws ssm describe-automation-executions`: LOW — returns execution metadata
- `aws ssm describe-instance-information`: MEDIUM — returns instance details and agent status
- `aws cloudtrail lookup-events`: MEDIUM — reveals IAM principals and actions
- `aws service-quotas list-service-quotas`: LOW — returns quota values

## Prohibited Actions

- NEVER make changes to production automation without understanding the root cause first
- NEVER disable SSM features to work around issues
- NEVER ignore recurring automation failures
- NEVER modify IAM permissions broadly to fix specific automation issues

## Phase 3 — Rollback

1. Revert any diagnostic changes made during troubleshooting
2. Restore original automation configurations if modified
3. Verify all automation workflows function correctly after troubleshooting
4. Document findings and share with the team
5. Schedule a review if the root cause was not fully identified

## Common Issues

- symptoms: "All automations suddenly failing across the account"
  diagnosis: "IAM policy change, SCP update, or SSM service issue."
  resolution: "Check recent IAM/SCP changes. Verify SSM service health. Review CloudTrail for policy modifications."

- symptoms: "Automation works in one region but not another"
  diagnosis: "Document not deployed to the target region, or regional IAM/service differences."
  resolution: "Verify document exists in the target region. Check regional service availability and IAM configuration."

- symptoms: "Automation execution count approaching quota limits"
  diagnosis: "Too many concurrent or queued automation executions."
  resolution: "Request quota increase. Implement execution queuing. Review if all automations are necessary."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: general_error
    content: "<error details and diagnostic findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and preventive measures"
```

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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
