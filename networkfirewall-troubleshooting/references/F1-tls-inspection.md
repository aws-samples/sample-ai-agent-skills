---
title: "F1 — TLS Inspection Issues"
description: "Diagnose TLS inspection configuration and operation problems"
status: active
severity: HIGH
triggers:
  - "TLS inspection"
  - "HTTPS inspection"
  - "SSL decryption"
  - "TLS not working"
owner: devops-agent
objective: "Identify and resolve TLS inspection configuration issues"
context: "TLS inspection decrypts HTTPS traffic for stateful rule inspection and re-encrypts it. Requires a CA certificate in ACM, TLS inspection configuration on the firewall policy, and client trust of the CA certificate. Not all TLS versions and cipher suites are supported. TLS inspection impacts throughput and latency."
---

## Phase 1 — Triage

MUST:
- Check TLS inspection configuration on firewall policy: `aws network-firewall describe-tls-inspection-configuration --tls-inspection-configuration-name <name>`
- Verify CA certificate exists in ACM: `aws acm describe-certificate --certificate-arn <ca-cert-arn>`
- Check if TLS inspection is associated with firewall policy
- Verify clients trust the CA certificate

SHOULD:
- Check for TLS-related errors in alert logs
- Verify supported TLS versions and cipher suites
- Check certificate chain validity

MAY:
- Test with a specific HTTPS connection to isolate issues
- Check for certificate pinning in applications (breaks TLS inspection)

## Phase 2 — Remediate

MUST:
- Import CA certificate to ACM if missing
- Create TLS inspection configuration: `aws network-firewall create-tls-inspection-configuration --tls-inspection-configuration-name <name> --tls-inspection-configuration <config>`
- Deploy CA certificate to client trust stores

SHOULD:
- Exclude certificate-pinned applications from TLS inspection
- Configure scope (which traffic to inspect) to minimize impact

MAY:
- Implement CA certificate distribution automation
- Monitor TLS inspection performance impact

## Common Issues

- symptoms: "Certificate warnings in browsers"
  diagnosis: "Clients don't trust the CA certificate used for TLS inspection."
  resolution: "Deploy CA certificate to client trust stores (OS, browser, Java)."

- symptoms: "Application fails with TLS error"
  diagnosis: "Application uses certificate pinning."
  resolution: "Exclude the application's traffic from TLS inspection."

- symptoms: "TLS inspection not decrypting traffic"
  diagnosis: "TLS inspection configuration not associated with policy or scope too narrow."
  resolution: "Associate TLS config with firewall policy. Check inspection scope."

## Output Format

```yaml
root_cause: "tls_inspection — <specific_cause>"
evidence:
  - type: tls_config
    content: "<TLS inspection configuration>"
  - type: ca_certificate
    content: "<CA certificate status>"
  - type: client_trust
    content: "<client trust store status>"
severity: HIGH
mitigation:
  immediate: "Fix TLS inspection configuration and client trust"
  long_term: "Automate CA certificate distribution and monitor impact"
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
