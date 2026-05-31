---
title: "G1 — Encryption Issues"
description: "Diagnose KMS encryption configuration problems"
status: active
severity: HIGH
triggers: ["KMS error", "encryption failed", "key access denied"]
owner: devops-agent
objective: "Identify and resolve encryption issues"
context: "Encryption issues occur when: KMS key disabled, key policy wrong, cross-account key access missing, or key deleted."
---

## Phase 1 — Triage

MUST:
- Check KMS key status: `aws kms describe-key --key-id <key-id>`
- Verify key policy allows HealthLake service
- Check if key is enabled and not pending deletion
- Verify key is in same region as data store

SHOULD:
- Check key grants for HealthLake
- Verify cross-account key access if applicable
- Review CloudTrail for KMS access denied events

## Phase 2 — Remediate

MUST:
- Enable KMS key if disabled
- Fix key policy to allow HealthLake service principal
- Use AWS-managed key as fallback

SHOULD:
- Monitor KMS key usage and rotation
- Document key management procedures

## Output Format

```yaml
root_cause: "encryption — <specific_cause>"
evidence:
  - type: kms_key
    content: "<key status and policy>"
severity: HIGH
```

## Safety Ratings
```
safety_ratings:
  - "Triage commands (describe/list/get): GREEN — read-only"
  - "Configuration changes: YELLOW — recoverable"
  - "Delete/terminate operations: RED — irreversible"
```

## Escalation Conditions
- "Resource serves production workload"
- "Fix requires deleting or recreating resources"
- "Fix involves security or encryption changes"
- "Blast radius affects multiple dependent services"

## Data Sensitivity
- HIGH: credentials, PII, security configurations
- MEDIUM: resource configurations, metrics
- LOW: status information, error messages

## Prohibited Actions
- Never delete FHIR datastores containing patient health data without authorization
- Never modify encryption settings on active datastores without compliance review
- Never import untested or unvalidated FHIR data into production datastores
- Never disable access controls or audit logging on datastores with PHI
- Never export patient data without verifying HIPAA compliance requirements

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Restore previous datastore configuration from documented state if imports fail
- Revert access control changes immediately if unauthorized access is detected
- Re-enable encryption and audit settings if they were inadvertently modified

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
