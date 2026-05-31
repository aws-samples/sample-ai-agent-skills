---
title: "Z1 — General Network Manager Troubleshooting"
description: "Catch-all runbook for Network Manager and Cloud WAN issues"
status: active
severity: MEDIUM
triggers: ["Network Manager error", "Cloud WAN not working", "network manager issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Network Manager or Cloud WAN issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List global networks: `aws networkmanager describe-global-networks`
- List core networks: `aws networkmanager list-core-networks`
- List attachments: `aws networkmanager list-attachments --core-network-id <id>`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=networkmanager.amazonaws.com --max-results 20`

SHOULD:
- Check core network state (AVAILABLE, UPDATING, etc.)
- Verify the active policy version
- Check IAM permissions for the caller

MAY:
- Check AWS Health Dashboard for Network Manager service issues
- Review CloudWatch metrics for anomalies

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook (A1–G1)

SHOULD:
- Document findings and investigation path

MAY:
- Set up monitoring for the identified issue category

## Common Issues

- symptoms: "General Network Manager API errors"
  diagnosis: "IAM permissions, service quotas, or regional availability."
  resolution: "Check IAM permissions, verify service quotas, and ensure the API is called in a supported region."

## Safety Ratings

```
safety_ratings:
  - "describe-global-networks: GREEN — read-only enumeration"
  - "list-core-networks: GREEN — read-only enumeration"
  - "list-attachments: GREEN — read-only enumeration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Network Manager service appears non-functional across multiple regions
- Multiple unrelated issues occurring simultaneously
- CloudTrail shows unauthorized API calls to Network Manager
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws networkmanager describe-global-networks`: LOW — returns network metadata
- `aws networkmanager list-core-networks`: LOW — returns core network metadata
- `aws networkmanager list-attachments`: LOW — returns attachment metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without first identifying the root cause
- NEVER delete core networks, attachments, or global networks during investigation
- NEVER execute policy changes as a troubleshooting step without understanding the impact
- NEVER dismiss the issue without documenting the investigation path and findings

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them to the pre-investigation state
2. Verify core network state is AVAILABLE
3. Confirm all attachments are in expected states
4. Document all investigation steps and findings for the network team
5. Verify the Network Manager service is functioning normally before closing the investigation

## Output Format

```yaml
root_cause: "general_network_manager — <specific_cause>"
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
