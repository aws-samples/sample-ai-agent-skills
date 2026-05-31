---
title: "E2 — Driver Issues"
description: "Diagnose MongoDB driver compatibility and configuration issues with DocumentDB"
status: active
severity: MEDIUM
triggers:
  - "driver error"
  - "driver compatibility"
  - "connection string"
  - "driver version"
  - "pymongo error"
  - "mongoose error"
owner: devops-agent
objective: "Identify and resolve MongoDB driver issues when connecting to DocumentDB"
context: "DocumentDB works with standard MongoDB drivers but requires specific configuration for TLS, connection strings, and read preferences. Driver version compatibility, TLS configuration, and connection string format are common sources of issues."
---

## Phase 1 — Triage

MUST:
- Identify the driver and version being used
- Check the connection string format and parameters
- Verify TLS configuration in the driver
- Check for driver-specific error messages

SHOULD:
- Verify driver version is compatible with the DocumentDB engine version
- Check if the driver supports the required MongoDB wire protocol version
- Review driver connection options (retryWrites, readPreference, etc.)
- Test with the mongo shell to isolate driver vs server issues

MAY:
- Check driver release notes for known DocumentDB issues
- Review driver debug/verbose logging
- Compare driver behavior with different DocumentDB engine versions

## Phase 2 — Remediate

MUST:
- Use a driver version compatible with the DocumentDB engine version
- Configure TLS correctly in the driver (CA certificate, TLS enabled)
- Set retryWrites=false (DocumentDB does not support retryable writes in all versions)

SHOULD:
- Use the latest stable driver version
- Configure appropriate read preference (primaryPreferred or secondaryPreferred)
- Set connection pool parameters in the driver

MAY:
- Implement driver-level monitoring and logging
- Test driver upgrades in a staging environment

## Common Issues

- symptoms: "retryWrites error or unexpected behavior"
  diagnosis: "DocumentDB 3.6/4.0 does not support retryable writes. Some drivers enable it by default."
  resolution: "Set retryWrites=false in the connection string. DocumentDB 5.0 has limited retry support."

- symptoms: "Driver authentication failure despite correct credentials"
  diagnosis: "Driver is using an authentication mechanism not supported by DocumentDB."
  resolution: "Use SCRAM-SHA-1 authentication. Set authMechanism=SCRAM-SHA-1 in the connection string."

- symptoms: "Driver cannot parse DocumentDB connection string"
  diagnosis: "Connection string format is incorrect or uses unsupported options."
  resolution: "Use the connection string from the DocumentDB console. Format: mongodb://<user>:<pass>@<endpoint>:27017/?tls=true&tlsCAFile=<path>&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false"

## Output Format

```yaml
root_cause: "driver_issue — <specific_cause>"
evidence:
  - type: driver
    content: "<driver name, version, and configuration>"
  - type: connection_string
    content: "<connection string analysis>"
  - type: error
    content: "<driver error messages>"
severity: MEDIUM
mitigation:
  immediate: "Fix driver configuration or upgrade driver version"
  long_term: "Standardize driver configuration across applications"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | MEDIUM — Driver configuration includes connection strings and credentials |
| Blast Radius | MEDIUM — Driver changes affect all application instances |
| Rollback Complexity | LOW — Driver version and configuration can be reverted |
| Compliance Scope | LOW — Driver configuration is operational |

## Escalation Conditions

- Escalate to application teams if driver changes cause connectivity failures
- Escalate to security if driver configuration exposes credentials

## Data Sensitivity

- **Connection strings** may contain credentials — never log or share
- **Driver configuration** reveals database access patterns

## PROHIBITED Actions

- Do NOT store connection strings with credentials in version control
- Do NOT upgrade drivers in production without testing in staging
- Do NOT enable retryWrites on DocumentDB 3.6/4.0 — it's not supported

## Phase 3 — Rollback

- Revert driver version to previous known-working version
- Restore previous connection string configuration
- Redeploy application with reverted driver settings

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
