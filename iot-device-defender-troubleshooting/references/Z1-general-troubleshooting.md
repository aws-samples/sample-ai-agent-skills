---
title: "Z1 — General IoT Device Defender Troubleshooting"
description: "Catch-all runbook for IoT Device Defender issues"
status: active
severity: MEDIUM
triggers: ["Device Defender error", "IoT security issue", "defender not working"]
owner: devops-agent
objective: "Systematic investigation of unclassified IoT Device Defender issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- Check audit configuration: `aws iot describe-account-audit-configuration`
- List security profiles: `aws iot list-security-profiles`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iot.amazonaws.com --max-results 20`

SHOULD:
- Check recent audit tasks: `aws iot list-audit-tasks --task-status COMPLETED --start-time <7-days-ago> --end-time <now>`
- Check active violations: `aws iot list-active-violations`

MAY:
- Check AWS Health Dashboard for IoT service issues
- Review service quotas for Device Defender

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
  - "iot describe-account-audit-configuration: GREEN — read-only configuration check"
  - "iot list-security-profiles: GREEN — read-only listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "iot list-active-violations: GREEN — read-only violation listing"
```

## Escalation Conditions

- Device Defender service appears non-functional in the region
- Multiple unrelated features (Audit and Detect) fail simultaneously
- Issue cannot be classified after systematic investigation
- Service quota limits are blocking operations

## Data Sensitivity

- `aws iot describe-account-audit-configuration`: LOW — returns audit configuration
- `aws iot list-security-profiles`: LOW — returns profile names
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- `aws iot list-active-violations`: MEDIUM — reveals device security status

## Prohibited Actions

- NEVER make configuration changes during general troubleshooting without identifying root cause
- NEVER disable audit or Detect as a troubleshooting step
- NEVER dismiss the issue without documenting the investigation path
- NEVER share violation data containing device identifiers in public channels

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore any resources that were modified during troubleshooting
3. Verify Device Defender audit and Detect are functioning normally
4. Document all investigation steps and findings
5. Verify the issue is resolved or properly escalated

## Common Issues

- symptoms: "General Device Defender issues — multiple features not working"
  diagnosis: "Often caused by IAM permission issues or service not configured."
  resolution: "Verify IAM permissions and check audit/Detect configuration."

## Output Format

```yaml
root_cause: "general_device_defender — <specific_cause>"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
