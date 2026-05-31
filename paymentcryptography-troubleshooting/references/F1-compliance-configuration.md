---
title: "F1 — Compliance Configuration"
description: "Diagnose PCI DSS compliance configuration issues"
status: active
severity: CRITICAL
triggers: ["PCI compliance", "compliance error", "compliance configuration", "PCI DSS"]
owner: devops-agent
objective: "Resolve PCI DSS compliance configuration issues for Payment Cryptography"
context: "Payment Cryptography is designed for PCI DSS compliance. Key management must follow PCI DSS requirements including dual control and split knowledge."
---

## Phase 1 — Triage

MUST:
- Verify key management follows dual control: check IAM roles separate key creation from key usage
- Check audit logging is comprehensive: `aws cloudtrail describe-trails`
- Verify key rotation policies are in place

SHOULD:
- Check key expiration dates and rotation schedules
- Verify access control follows least privilege
- Review key export policies

MAY:
- Audit key usage patterns for anomalies
- Review compliance documentation completeness

## Phase 2 — Remediate

MUST:
- Implement dual control for key management (separate roles for creation and usage)
- Enable comprehensive audit logging (control plane and data plane)
- Configure key rotation schedules per PCI DSS requirements

SHOULD:
- Implement key expiration monitoring
- Document key management procedures
- Set up compliance monitoring dashboards

MAY:
- Automate compliance checks with AWS Config rules
- Implement automated key rotation workflows

## Safety Ratings

```
safety_ratings:
  - "cloudtrail describe-trails: GREEN — read-only trail check"
  - "IAM role review: GREEN — read-only permission analysis"
  - "Key rotation configuration: YELLOW — changes key lifecycle"
  - "Compliance remediation: YELLOW — modifies security configuration"
```

## Escalation Conditions

- PCI DSS audit finding requires immediate remediation
- Key management procedures violate dual control requirements
- Audit logging gaps during compliance assessment period
- Key rotation has not been performed within required timeframe

## Data Sensitivity

- Compliance configuration: MEDIUM — reveals security posture
- Key management procedures: MEDIUM — describes operational processes
- Audit findings: HIGH — reveals compliance gaps
- Key rotation schedules: LOW — operational metadata

## Prohibited Actions

- NEVER bypass dual control requirements for key management
- NEVER disable audit logging to reduce costs
- NEVER skip key rotation to avoid operational disruption
- NEVER share compliance audit findings outside authorized personnel

## Phase 3 — Rollback

1. If compliance configuration was changed incorrectly: revert to previous settings
2. If IAM roles were modified: restore dual control separation
3. If audit logging was disrupted: restore immediately and document the gap
4. Verify all compliance controls are functioning after changes
5. Document all changes for the next PCI DSS audit

## Common Issues

- symptoms: "PCI DSS audit finding: no dual control for key management"
  diagnosis: "Single IAM role has both key creation and key usage permissions."
  resolution: "Create separate IAM roles: one for key management (control plane) and one for cryptographic operations (data plane)."

- symptoms: "PCI DSS audit finding: key rotation not performed"
  diagnosis: "Keys have not been rotated within the required period."
  resolution: "Implement key rotation: create new key, migrate operations, schedule old key for deletion."

## Output Format

```yaml
root_cause: "compliance — <specific_cause>"
evidence:
  - type: compliance_finding
    content: "<finding details>"
severity: CRITICAL
mitigation:
  immediate: "Address compliance gap"
  long_term: "Implement automated compliance monitoring"
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
