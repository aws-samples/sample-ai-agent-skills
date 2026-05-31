---
title: "E1 — Certificate Management Issues"
description: "Diagnose certificate lifecycle management problems"
status: active
severity: HIGH
triggers:
  - "certificate rotation"
  - "certificate expiring"
  - "CA certificate"
  - "certificate provisioning"
  - "JITP issues"
owner: devops-agent
objective: "Identify and resolve certificate management issues"
context: "Certificate management issues include: certificate rotation failures, CA certificate registration, just-in-time provisioning (JITP) errors, certificate transfer problems, or bulk certificate operations."
---

## Phase 1 — Triage

MUST:
- List certificates: `aws iot list-certificates --query 'certificates[*].{Id:certificateId,Status:status,Created:creationDate}'`
- Check CA certificates: `aws iot list-ca-certificates`
- Check certificate details: `aws iot describe-certificate --certificate-id <cert-id>`
- Verify JITP template if using just-in-time provisioning

SHOULD:
- Check certificate expiration dates
- Verify CA certificate is ACTIVE
- Check provisioning template: `aws iot describe-provisioning-template --template-name <name>`

MAY:
- Review certificate transfer status
- Check for orphaned certificates (not attached to any thing)

## Phase 2 — Remediate

MUST:
- Rotate expiring certificates before they expire
- Register CA certificate: `aws iot register-ca-certificate --ca-certificate file://ca.pem --verification-certificate file://verify.pem --set-as-active`
- Fix JITP template if provisioning fails

SHOULD:
- Implement automated certificate rotation
- Set up certificate expiration monitoring
- Use fleet provisioning for scalable certificate management

MAY:
- Implement certificate revocation for compromised devices
- Use AWS IoT Device Defender for certificate auditing

## Output Format

```yaml
root_cause: "certificate_management_issue — <specific_cause>"
evidence:
  - type: cert_lifecycle
    content: "<certificate status and expiration>"
  - type: ca_status
    content: "<CA certificate status>"
severity: HIGH
mitigation:
  immediate: "Fix certificate status or rotation"
  long_term: "Implement automated certificate lifecycle management"
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
