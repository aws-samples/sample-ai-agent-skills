---
title: "B2 — TLS Issues"
description: "Diagnose TLS/SSL connection and certificate problems"
status: active
severity: HIGH
triggers:
  - "TLS error"
  - "SSL handshake failed"
  - "certificate error"
  - "SSL connection"
  - "TLS version"
owner: devops-agent
objective: "Identify and resolve TLS connection issues"
context: "TLS issues include: client not supporting required TLS version, certificate trust issues, wrong port (non-TLS vs TLS), or cipher suite mismatch."
---

## Phase 1 — Triage

MUST:
- Verify using TLS port (61617 for ActiveMQ OpenWire, 5671 for RabbitMQ AMQP)
- Check client TLS version support (TLS 1.2 minimum)
- Verify client trusts Amazon's CA certificates
- Check broker TLS configuration

SHOULD:
- Test TLS connection: `openssl s_client -connect <endpoint>:<port>`
- Verify cipher suite compatibility
- Check if client library supports required TLS version

MAY:
- Check for intermediate certificate issues
- Review client-side TLS configuration

## Phase 2 — Remediate

MUST:
- Use TLS-enabled ports for all connections
- Ensure client supports TLS 1.2 or higher
- Install Amazon Root CA certificates on client

SHOULD:
- Update client libraries to support modern TLS
- Use connection strings with ssl:// or amqps:// scheme

MAY:
- Configure specific cipher suites if needed
- Implement certificate pinning for additional security

## Output Format

```yaml
root_cause: "tls_issue — <specific_cause>"
evidence:
  - type: tls_test
    content: "<TLS handshake result>"
  - type: port_config
    content: "<port and protocol>"
severity: HIGH
mitigation:
  immediate: "Fix TLS version, port, or certificate trust"
  long_term: "Standardize TLS configuration across all clients"
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

- **NEVER** force-reboot a production broker during active message processing without draining connections
- **NEVER** delete a broker with persistent messages that have not been consumed or backed up
- **NEVER** modify authentication credentials on production brokers without coordinating with all connected clients
- **NEVER** change network-of-brokers topology during peak message throughput periods
- **NEVER** downgrade broker engine versions or instance types on production brokers without capacity testing

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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
