---
title: "F2 — Certificate Management Issues"
description: "Diagnose certificate management problems for TLS inspection"
status: active
severity: MEDIUM
triggers:
  - "certificate error"
  - "CA certificate"
  - "certificate expired"
  - "certificate management"
owner: devops-agent
objective: "Identify and resolve certificate management issues for TLS inspection"
context: "TLS inspection requires a CA certificate imported into ACM. The CA certificate is used to generate server certificates on-the-fly during inspection. Certificate issues include expiration, import failures, key format problems, and trust chain issues."
---

## Phase 1 — Triage

MUST:
- Check CA certificate in ACM: `aws acm describe-certificate --certificate-arn <ca-cert-arn>`
- Verify certificate is not expired
- Check certificate type (must be imported, not ACM-issued)
- Verify private key is available (required for CA operations)

SHOULD:
- Check certificate chain completeness
- Verify key algorithm is supported (RSA 2048+, ECDSA P-256/P-384)
- Check TLS inspection configuration references correct certificate

MAY:
- Verify certificate validity period has sufficient remaining time
- Check for certificate revocation

## Phase 2 — Remediate

MUST:
- Import CA certificate to ACM: `aws acm import-certificate --certificate fileb://ca-cert.pem --private-key fileb://ca-key.pem --certificate-chain fileb://ca-chain.pem`
- Ensure certificate is valid and not expired
- Update TLS inspection configuration if certificate ARN changed

SHOULD:
- Set up certificate expiration monitoring
- Plan certificate rotation before expiration

MAY:
- Use AWS Private CA for managed CA certificate lifecycle
- Implement automated certificate rotation

## Common Issues

- symptoms: "Certificate import fails"
  diagnosis: "Invalid PEM format, key mismatch, or unsupported algorithm."
  resolution: "Verify PEM format. Ensure private key matches certificate. Use RSA 2048+."

- symptoms: "TLS inspection fails after certificate expiration"
  diagnosis: "CA certificate expired in ACM."
  resolution: "Import renewed CA certificate. Update TLS inspection configuration."

- symptoms: "Generated certificates not trusted"
  diagnosis: "CA certificate not in client trust stores or chain incomplete."
  resolution: "Deploy CA certificate to clients. Include full certificate chain."

## Output Format

```yaml
root_cause: "certificate_management — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<CA certificate details and validity>"
  - type: tls_config
    content: "<TLS inspection certificate reference>"
severity: MEDIUM
mitigation:
  immediate: "Fix certificate issue (import, renew, or update)"
  long_term: "Implement certificate lifecycle management and monitoring"
```


## Safety Ratings

| Aspect | Rating | Notes |
|--------|--------|-------|
| Data Risk | Medium | Operations may affect service configurations and data access patterns |
| Service Impact | High | Incorrect changes can disrupt active workloads and dependent services |
| Recovery Complexity | Medium | Most actions are reversible but may require coordinated rollback procedures |
| Blast Radius | Medium | Impact typically scoped to the affected resource and its direct dependents |

## Escalation Conditions

- **Escalate immediately** if the issue involves potential data loss or corruption
- **Escalate immediately** if a production workload is impacted and cannot be restored within the agreed SLA
- **Escalate immediately** if the root cause is unclear after initial diagnostic steps are exhausted
- **Escalate to service team** if the issue appears to be a platform-level bug or undocumented behavior
- **Escalate to security team** if unauthorized access, policy violations, or credential exposure is suspected

## Data Sensitivity

- Verify data classification before performing any read or export operations
- Ensure all troubleshooting outputs (logs, configurations, snapshots) are handled according to the customer's data handling policy
- Mask or redact personally identifiable information (PII) in all shared diagnostics
- Do not copy production data to non-production environments without explicit authorization
- Retain troubleshooting artifacts only for the duration required and dispose of them securely

## Prohibited Actions

- **NEVER** delete or modify firewall rules that are actively filtering production traffic without a tested rollback plan
- **NEVER** change route table associations for firewall endpoints without verifying traffic flow continuity
- **NEVER** disable TLS inspection on endpoints monitoring compliance-regulated traffic
- **NEVER** remove stateful rule groups from a firewall policy during active threat mitigation
- **NEVER** modify asymmetric routing configurations without confirming bidirectional traffic inspection

## Phase 3 � Rollback

1. **Identify rollback trigger**: Determine if the applied change has caused degradation, unexpected behavior, or failed validation checks
2. **Preserve current state**: Capture logs, configuration snapshots, and resource states before initiating rollback
3. **Execute rollback steps**: Revert configuration changes in reverse order of application, verifying each step
4. **Validate service recovery**: Confirm that the service has returned to its pre-change operational state using health checks and functional tests
5. **Document rollback**: Record the rollback trigger, steps taken, time to recovery, and any residual issues for post-incident review
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
  - "NEVER suggest disabling stateful inspection"
  - "NEVER suggest removing all firewall rules"
  - "NEVER suggest bypassing TLS inspection without review"
