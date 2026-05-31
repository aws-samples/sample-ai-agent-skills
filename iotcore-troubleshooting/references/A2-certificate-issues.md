---
title: "A2 — Certificate Issues"
description: "Diagnose X.509 certificate authentication problems"
status: active
severity: HIGH
triggers:
  - "certificate error"
  - "certificate expired"
  - "certificate inactive"
  - "authentication failed"
  - "TLS certificate"
owner: devops-agent
objective: "Identify and resolve certificate authentication issues"
context: "Certificate issues include: certificate not active, expired, not attached to thing/policy, wrong CA, certificate revoked, or certificate ID mismatch."
---

## Phase 1 — Triage

MUST:
- Check certificate status: `aws iot describe-certificate --certificate-id <cert-id>`
- Check certificate attachments: `aws iot list-thing-principals --thing-name <thing-name>`
- Check policies on certificate: `aws iot list-attached-policies --target <cert-arn>`
- Verify certificate is not expired (check validity dates)

SHOULD:
- Verify certificate was signed by a registered CA or AWS IoT CA
- Check if certificate was recently revoked
- Verify certificate chain is complete (device cert + CA cert)

MAY:
- Check CRL (Certificate Revocation List) status
- Verify certificate fingerprint matches expected value

## Phase 2 — Remediate

MUST:
- Activate inactive certificate: `aws iot update-certificate --certificate-id <cert-id> --new-status ACTIVE`
- Attach to thing if missing: `aws iot attach-thing-principal --thing-name <thing> --principal <cert-arn>`
- Attach policy if missing: `aws iot attach-policy --policy-name <policy> --target <cert-arn>`

SHOULD:
- Rotate expired certificates with new ones
- Register CA certificate if using custom CA: `aws iot register-ca-certificate --ca-certificate file://ca.pem --verification-certificate file://verify.pem`

MAY:
- Implement automated certificate rotation
- Set up certificate expiration monitoring

## Output Format

```yaml
root_cause: "certificate_issue — <specific_cause>"
evidence:
  - type: cert_status
    content: "<certificate status and details>"
  - type: attachments
    content: "<thing and policy attachments>"
severity: HIGH
mitigation:
  immediate: "Fix certificate status or attachments"
  long_term: "Implement certificate lifecycle management"
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
- Never revoke or delete device certificates without confirming device decommission
- Never modify IoT policies on production devices without testing in a staging environment
- Never delete thing groups that contain active production devices
- Never alter MQTT topic rules that route critical telemetry data
- Never disable custom authorizers serving active device connections

## Phase 3 — Rollback
- "Document current state before any changes"
- "If fix worsens the issue, revert to documented state"
- Reattach previous certificates if device connectivity is lost after changes
- Restore previous IoT policy versions if devices lose permissions
- Revert rule engine action configurations if message routing breaks

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
