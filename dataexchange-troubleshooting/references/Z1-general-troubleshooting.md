---
title: "Z1 — General Data Exchange Troubleshooting"
description: "Catch-all runbook for Data Exchange issues"
status: active
severity: MEDIUM
triggers: ["Data Exchange error", "dataexchange not working", "data exchange issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Data Exchange issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List data sets: `aws dataexchange list-data-sets`
- Check recent jobs: `aws dataexchange list-jobs --max-results 10`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=dataexchange.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions for Data Exchange operations
- Verify the correct AWS region is being used

MAY:
- Check AWS Health Dashboard for service issues
- Review service quotas

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for identified issue area

## Safety Ratings

```
safety_ratings:
  - "dataexchange list-data-sets: GREEN — read-only enumeration"
  - "dataexchange list-jobs: GREEN — read-only job listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Data Exchange service appears non-functional in the region
- Multiple unrelated operations fail simultaneously
- Issue cannot be classified after systematic investigation
- Service quota limits are blocking operations

## Data Sensitivity

- `aws dataexchange list-data-sets`: LOW — returns data set metadata
- `aws dataexchange list-jobs`: LOW — returns job metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER delete data sets or revisions as a troubleshooting step
- NEVER dismiss the issue without documenting the investigation path
- NEVER share CloudTrail logs containing account details in public channels

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore any resources that were modified during troubleshooting
3. Verify Data Exchange operations are functioning normally
4. Document all investigation steps and findings
5. Verify the issue is resolved or properly escalated

## Common Issues

- symptoms: "General Data Exchange errors across multiple operations"
  diagnosis: "Often caused by IAM permission issues or incorrect region."
  resolution: "Verify IAM permissions and region configuration first."

## Output Format

```yaml
root_cause: "general_dataexchange — <specific_cause>"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
