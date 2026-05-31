---
title: "Z1 — General Braket Troubleshooting"
description: "Catch-all runbook for Amazon Braket issues"
status: active
severity: MEDIUM
triggers: ["Braket error", "braket not working", "quantum computing issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Braket issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check recent tasks: `aws braket search-quantum-tasks --filters '[{"name":"createdAfter","operator":"EQUAL","values":["<date>"]}]'`
- List available devices: `aws braket search-devices`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=braket.amazonaws.com --max-results 20`

SHOULD:
- Check IAM permissions for the Braket service
- Verify AWS region supports Braket

MAY:
- Check AWS Health Dashboard for Braket service issues

## Phase 2 — Remediate

MUST:
- Identify problem domain and pivot to specific runbook

SHOULD:
- Document findings and investigation path

MAY:
- Set up CloudWatch monitoring for Braket resources

## Safety Ratings

```
safety_ratings:
  - "search-quantum-tasks: GREEN — read-only task enumeration"
  - "search-devices: GREEN — read-only device listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Braket service appears non-functional in the region
- No devices available across all regions
- CloudTrail shows unauthorized API calls to Braket
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws braket search-quantum-tasks`: LOW — returns task metadata
- `aws braket search-devices`: LOW — returns device metadata
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER cancel running tasks or jobs during general troubleshooting without confirmation
- NEVER modify IAM policies as a first troubleshooting step
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path and findings

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore IAM policies if they were modified during troubleshooting
3. Verify no tasks or jobs were disrupted by investigation activities
4. Document all investigation steps and findings for the team
5. Verify the Braket service is functioning normally before closing the investigation

## Common Issues

- symptoms: "General 'access denied' errors when using Braket"
  diagnosis: "IAM policy missing braket:* permissions."
  resolution: "Attach AmazonBraketFullAccess managed policy or create custom policy with required braket actions."

- symptoms: "Braket API calls return 'service not available' errors"
  diagnosis: "Region does not support Amazon Braket."
  resolution: "Switch to a supported region: us-east-1, us-west-1, us-west-2, or eu-west-2."

## Output Format

```yaml
root_cause: "general_braket — <specific_cause>"
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
