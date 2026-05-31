---
title: "Z1 — General SSM Distributor Troubleshooting"
description: "Catch-all runbook for SSM Distributor issues"
status: active
severity: MEDIUM
triggers: ["Distributor error", "Distributor not working", "package distribution problem"]
owner: devops-agent
objective: "Systematic investigation of unclassified SSM Distributor issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List packages: `aws ssm list-documents --document-filter-list key=DocumentType,value=Package --query 'DocumentIdentifiers[*].{Name:Name,Owner:Owner}'`
- Check managed instances: `aws ssm describe-instance-information --query 'InstanceInformationList[*].{InstanceId:InstanceId,PingStatus:PingStatus,AgentVersion:AgentVersion}'`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=ssm.amazonaws.com --max-results 20`

SHOULD:
- Check recent command history: `aws ssm list-commands --query 'Commands[*].{CommandId:CommandId,DocumentName:DocumentName,Status:Status}'`
- Verify SSM service endpoint connectivity

MAY:
- Check AWS Health Dashboard for SSM service issues
- Review CloudWatch metrics for SSM

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for recurring issues

## Safety Ratings

```
safety_ratings:
  - "list-documents: GREEN — read-only package enumeration"
  - "describe-instance-information: GREEN — read-only instance metadata"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "list-commands: GREEN — read-only command history"
```

## Escalation Conditions

- SSM Distributor service appears non-functional in the region
- Multiple unrelated Distributor operations fail simultaneously
- CloudTrail shows unauthorized API calls to SSM
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws ssm list-documents`: LOW — returns document metadata
- `aws ssm describe-instance-information`: LOW — returns instance metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- `aws ssm list-commands`: MEDIUM — reveals command history and targets

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without first identifying the root cause
- NEVER delete packages, associations, or documents as a troubleshooting step
- NEVER dismiss the issue without documenting the investigation path and findings
- NEVER run commands on production instances without understanding their impact

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to the pre-investigation state
2. Restore any modified associations, packages, or IAM policies
3. Verify all Distributor operations are functioning normally
4. Document all investigation steps and findings for the operations team
5. Verify managed instances are online and responsive before closing the investigation

## Output Format

```yaml
root_cause: "general_distributor — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring and alerting"
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
