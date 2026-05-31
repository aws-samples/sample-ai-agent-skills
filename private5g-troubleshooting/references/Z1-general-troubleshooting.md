---
title: "Z1 — General Private 5G Troubleshooting"
description: "Catch-all runbook for Private 5G issues"
status: active
severity: MEDIUM
triggers: ["Private 5G error", "private 5G not working", "private network issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Private 5G issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List networks: `aws private-networks list-networks`
- Check network status: `aws private-networks get-network --network-arn <arn>`
- List network resources: `aws private-networks list-network-resources --network-arn <arn>`

SHOULD:
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=private-networks.amazonaws.com --max-results 20`
- Check CloudWatch metrics for anomalies

MAY:
- Check AWS Health Dashboard for service issues

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
  - "list-network-resources: GREEN — read-only resource listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Private 5G service appears non-functional in the region
- Multiple network components failing simultaneously
- CloudTrail shows unauthorized API calls to Private 5G
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws private-networks list-networks`: LOW — returns network metadata
- `aws private-networks list-network-resources`: LOW — returns resource metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER suggest deleting the network as a first troubleshooting step
- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER dismiss the issue without documenting the investigation path
- NEVER share network ARNs or resource identifiers in public channels

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Verify network status returns to AVAILABLE
3. Confirm all radio units are ACTIVE
4. Verify device connectivity is restored
5. Document all investigation steps and findings

## Common Issues

- symptoms: "General Private 5G error with no specific error message"
  diagnosis: "Issue may span multiple domains — network, radio, SIM, or VPC."
  resolution: "Follow triage steps systematically. Check network → sites → resources → devices → VPC."

## Output Format

```yaml
root_cause: "general_private5g — <specific_cause>"
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
