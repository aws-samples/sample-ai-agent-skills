---
title: "C2 — TLS Configuration Issues"
description: "Diagnose TLS/SSL certificate and configuration problems with DocumentDB"
status: active
severity: HIGH
triggers:
  - "TLS error"
  - "SSL error"
  - "certificate error"
  - "SSL handshake"
  - "certificate expired"
owner: devops-agent
objective: "Identify and resolve TLS configuration issues for DocumentDB connections"
context: "DocumentDB requires TLS by default. Issues include certificate bundle problems, certificate rotation, driver TLS configuration, and parameter group TLS settings. The rds-combined-ca-bundle.pem must be used for certificate verification."
---

## Phase 1 — Triage

MUST:
- Check TLS parameter in cluster parameter group: `aws docdb describe-db-cluster-parameters --db-cluster-parameter-group-name <param-group> --filters Name=ParameterName,Values=tls`
- Verify the certificate bundle is accessible on the client
- Check the connection string includes TLS parameters
- Verify cluster endpoint resolves correctly

SHOULD:
- Test TLS connection: `openssl s_client -connect <endpoint>:27017 -CAfile rds-combined-ca-bundle.pem`
- Check certificate expiration dates
- Verify driver TLS configuration matches DocumentDB requirements
- Check for certificate rotation events: `aws docdb describe-events --source-type db-cluster --duration 10080`

MAY:
- Verify intermediate certificate chain
- Check for driver version compatibility with TLS 1.2
- Review application TLS error logs

## Phase 2 — Remediate

MUST:
- Download the latest rds-combined-ca-bundle.pem from AWS
- Configure the connection string with --tls and --tlsCAFile
- Ensure the driver supports TLS 1.2 (minimum required)

SHOULD:
- Automate certificate bundle updates in deployment pipelines
- Test TLS connectivity before deploying application changes
- Monitor certificate expiration dates

MAY:
- Configure certificate pinning if required by security policy
- Set up alerts for upcoming certificate rotations

## Common Issues

- symptoms: "SSL handshake failed — certificate verify failed"
  diagnosis: "The CA certificate bundle is missing, outdated, or incorrect."
  resolution: "Download the latest rds-combined-ca-bundle.pem from https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem."

- symptoms: "Connection works without TLS but fails with TLS"
  diagnosis: "Driver TLS configuration is incorrect or certificate path is wrong."
  resolution: "Verify tlsCAFile path is correct and the file is readable. Check driver TLS options."

- symptoms: "TLS connection fails after certificate rotation"
  diagnosis: "Application is using an old certificate bundle that does not include the new CA."
  resolution: "Update to the latest rds-combined-ca-bundle.pem which includes both old and new CAs."

## Output Format

```yaml
root_cause: "tls_issue — <specific_cause>"
evidence:
  - type: tls_config
    content: "<TLS parameter and connection string>"
  - type: certificate
    content: "<certificate verification results>"
  - type: driver
    content: "<driver TLS configuration>"
severity: HIGH
mitigation:
  immediate: "Fix certificate bundle or TLS configuration"
  long_term: "Automate certificate management and monitoring"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | CRITICAL — TLS protects data in transit including credentials |
| Blast Radius | HIGH — TLS misconfiguration affects all client connections |
| Rollback Complexity | LOW — TLS settings and certificates can be updated |
| Compliance Scope | CRITICAL — TLS is required for compliance in most frameworks |

## Escalation Conditions

- Escalate to security if TLS is disabled on production clusters
- Escalate to operations if certificate rotation breaks application connectivity
- Escalate to compliance if TLS configuration doesn't meet minimum standards

## Data Sensitivity

- **TLS certificates** protect data in transit — manage lifecycle carefully
- **Connection strings** may contain credentials — never log in plain text
- **Certificate bundles** are public but must be kept current

## PROHIBITED Actions

- Do NOT disable TLS on production clusters
- Do NOT use TLS versions below 1.2
- Do NOT ignore certificate expiration warnings
- Do NOT store connection strings with credentials in version control

## Phase 3 — Rollback

- Restore TLS parameter to "enabled" if it was disabled
- Download and deploy the latest certificate bundle
- Update application connection strings with correct TLS configuration
- Verify all clients can connect with TLS after changes

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
