---
title: "Z1 — General Wickr Troubleshooting"
description: "Catch-all runbook for Wickr issues"
status: active
severity: MEDIUM
triggers: ["Wickr error", "Wickr not working", "AWS Wickr issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Wickr issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List networks: `aws wickr list-networks`
- Check network status: `aws wickr get-network --network-id <id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=wickr.amazonaws.com --max-results 20`

SHOULD:
- Check CloudWatch metrics for anomalies
- Verify IAM permissions for Wickr operations

MAY:
- Check AWS Health Dashboard for Wickr service issues

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for the identified issue type

## Safety Ratings

```
safety_ratings:
  - "list-networks: GREEN — read-only enumeration"
  - "get-network: GREEN — read-only network details"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Wickr service appears non-functional in the region
- Multiple network components failing simultaneously
- CloudTrail shows unauthorized API calls to Wickr
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws wickr list-networks`: LOW — returns network metadata
- `aws wickr get-network`: LOW — returns network configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER suggest deleting the network as a first troubleshooting step
- NEVER attempt to access message content for troubleshooting
- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Verify network status returns to active
3. Confirm user connectivity is restored
4. Verify message delivery is functioning normally
5. Document all investigation steps and findings

## Common Issues

- symptoms: "General Wickr error with no specific error message"
  diagnosis: "Issue may span multiple domains — network, auth, messaging, or compliance."
  resolution: "Follow triage steps systematically. Check network → admin access → users → messaging → compliance."

## Output Format

```yaml
root_cause: "general_wickr — <specific_cause>"
evidence:
  - type: investigation_summary
    content: "<findings>"
severity: MEDIUM
mitigation:
  immediate: "Address identified issue"
  long_term: "Implement comprehensive monitoring"
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
