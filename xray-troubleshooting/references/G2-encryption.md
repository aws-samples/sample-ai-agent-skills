---
title: "G2 — Encryption Issues"
description: "Diagnose X-Ray encryption configuration problems"
status: active
severity: MEDIUM
triggers:
  - "X-Ray encryption"
  - "KMS key error"
  - "encryption configuration"
  - "trace encryption"
owner: devops-agent
objective: "Identify and resolve X-Ray encryption configuration issues"
context: "X-Ray encrypts trace data at rest. By default, it uses an AWS-managed key. You can configure a customer-managed KMS key for additional control. KMS key permissions and key policies must allow X-Ray to use the key."
---

## Phase 1 — Triage

MUST:
- Check current encryption config: `aws xray get-encryption-config`
- If using CMK, verify the KMS key exists and is enabled: `aws kms describe-key --key-id <key-id>`
- Check KMS key policy allows X-Ray service: look for xray.amazonaws.com in the key policy
- Verify the caller has kms:Encrypt and kms:Decrypt permissions

SHOULD:
- Check CloudTrail for KMS-related errors from X-Ray
- Verify the KMS key is in the same region as X-Ray
- Check for KMS key rotation status

MAY:
- Review KMS key grants for X-Ray
- Check KMS request quotas

## Phase 2 — Remediate

MUST:
- Fix KMS key policy to allow X-Ray:
  Add xray.amazonaws.com as a principal with kms:GenerateDataKey and kms:Decrypt
- Update encryption config: `aws xray put-encryption-config --type KMS --key-id <key-id>`
- Or revert to default encryption: `aws xray put-encryption-config --type NONE`

SHOULD:
- Use a customer-managed key for compliance requirements
- Enable key rotation on the CMK

MAY:
- Set up CloudWatch alarms for KMS errors
- Document encryption configuration

## Common Issues

- symptoms: "PutEncryptionConfig fails with KMS error"
  diagnosis: "KMS key policy doesn't allow X-Ray service."
  resolution: "Add xray.amazonaws.com to the KMS key policy with required permissions."

- symptoms: "Traces not being stored after changing encryption"
  diagnosis: "KMS key disabled or deleted."
  resolution: "Re-enable the KMS key or switch to a different key."

- symptoms: "Cannot read traces after encryption change"
  diagnosis: "Old traces encrypted with previous key that is no longer accessible."
  resolution: "Ensure the previous KMS key is still enabled for reading old traces."

## Output Format

```yaml
root_cause: "encryption_issue — <specific_cause>"
evidence:
  - type: encryption_config
    content: "<current encryption configuration>"
  - type: kms_key_status
    content: "<KMS key state and policy>"
  - type: permissions
    content: "<KMS permissions for X-Ray>"
severity: MEDIUM
mitigation:
  immediate: "Fix KMS key policy or encryption configuration"
  long_term: "Document encryption requirements and automate with IaC"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Trace data reveals application request flow and dependencies
- Segment metadata may contain request/response details
- Service maps expose complete application architecture
- Classification: CONFIDENTIAL - application architecture and request flow data

## Prohibited Actions

- NEVER modify sampling rules that could cause production trace data loss
- NEVER delete X-Ray groups used by production monitoring dashboards
- NEVER change encryption settings without coordinating with security team
- NEVER disable tracing on production services without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

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
