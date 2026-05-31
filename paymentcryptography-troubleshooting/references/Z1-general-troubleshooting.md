---
title: "Z1 — General Payment Cryptography Troubleshooting"
description: "Catch-all runbook for Payment Cryptography issues"
status: active
severity: MEDIUM
triggers: ["Payment Cryptography error", "payment-cryptography not working", "Payment Cryptography issue"]
owner: devops-agent
objective: "Systematic investigation of unclassified Payment Cryptography issues"
context: "Use when the issue doesn't clearly match a specific runbook."
---

## Phase 1 — Triage

MUST:
- List keys: `aws payment-cryptography list-keys`
- List aliases: `aws payment-cryptography list-aliases`
- Review CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=payment-cryptography.amazonaws.com --max-results 20`

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
  - "list-keys: GREEN — read-only enumeration"
  - "list-aliases: GREEN — read-only alias listing"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
  - "check AWS Health Dashboard: GREEN — read-only service status"
```

## Escalation Conditions

- Payment Cryptography service appears non-functional
- Multiple operation types (encrypt, PIN, MAC) fail simultaneously
- CloudTrail shows unauthorized key management operations
- Issue cannot be classified into any specific runbook after systematic investigation

## Data Sensitivity

- `aws payment-cryptography list-keys`: LOW — returns key metadata
- `aws payment-cryptography list-aliases`: LOW — returns alias mappings
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER delete keys or aliases during general troubleshooting
- NEVER perform cryptographic operations with production data during investigation
- NEVER make configuration changes without first identifying the root cause
- NEVER dismiss the issue without documenting the investigation path

## Phase 3 — Rollback

1. If any configuration changes were made during investigation, revert them
2. Restore key and alias settings if they were modified
3. If keys were disabled during investigation: re-enable them
4. Document all investigation steps and findings for PCI DSS audit
5. Verify the Payment Cryptography service is functioning normally before closing

## Output Format

```yaml
root_cause: "general_paymentcryptography — <specific_cause>"
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
