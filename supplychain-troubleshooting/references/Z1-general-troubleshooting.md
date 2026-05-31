---
title: "Z1 — General Supply Chain Troubleshooting"
description: "Catch-all runbook for Supply Chain issues"
status: active
severity: MEDIUM
triggers: ["Supply Chain error", "scn not working", "Supply Chain issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Supply Chain issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List instances: `aws scn list-instances`
- Check instance status: `aws scn get-instance --instance-id <instance-id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=scn.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions for the calling principal
- Verify service availability in the region

MAY:
- Check AWS Health Dashboard for service issues
- Review CloudWatch metrics for anomalies

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
  - "list-instances: GREEN — read-only enumeration"
  - "get-instance: GREEN — read-only instance details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Supply Chain service appears non-functional in the region
- Multiple components (planning, visibility, integration) fail simultaneously
- CloudTrail shows unauthorized API calls to Supply Chain
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws scn list-instances`: LOW — returns instance metadata
- `aws scn get-instance`: LOW — returns instance configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete instances or datasets during general troubleshooting
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER access business data without proper authorization

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore instance settings if they were modified
3. If data integration flows were paused: resume them
4. Document all investigation steps and findings
5. Verify the Supply Chain service is functioning normally before closing

## Output Format

```yaml
root_cause: "general_supplychain — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement monitoring"
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
