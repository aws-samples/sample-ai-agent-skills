---
title: "F2 — SSL/TLS Certificate Issues"
description: "Diagnose SSL/TLS certificate problems on Lightsail load balancers"
status: active
severity: HIGH
triggers:
  - "SSL error"
  - "certificate error"
  - "HTTPS not working"
  - "certificate validation"
owner: devops-agent
objective: "Identify and resolve SSL/TLS certificate issues on Lightsail load balancers"
context: "Lightsail load balancers include free SSL/TLS certificates managed by Lightsail. Certificates require DNS validation. Each LB supports one certificate with up to 10 alternate domain names. Certificates auto-renew. Custom certificates are not supported — use the Lightsail-managed certificate."
---

## Phase 1 — Triage

MUST:
- Check LB certificate: `aws lightsail get-load-balancer-tls-certificates --load-balancer-name <lb-name>`
- Verify certificate status (PENDING_VALIDATION, ISSUED, FAILED)
- Check DNS validation records are in place
- Verify domain names on certificate match actual domains

SHOULD:
- Check certificate expiration date
- Verify CNAME validation records in DNS
- Test HTTPS connectivity to the load balancer

MAY:
- Check for mixed content issues in the application
- Verify HTTP to HTTPS redirect is configured

## Phase 2 — Remediate

MUST:
- Create certificate: `aws lightsail create-load-balancer-tls-certificate --load-balancer-name <lb-name> --certificate-name <cert-name> --certificate-domain-name <domain> --certificate-alternative-names <alt-domains>`
- Add DNS validation CNAME records to your DNS zone
- Wait for validation to complete (can take minutes to hours)

SHOULD:
- Attach validated certificate to LB: `aws lightsail attach-load-balancer-tls-certificate --load-balancer-name <lb-name> --certificate-name <cert-name>`
- Enable HTTPS redirect

MAY:
- Add alternate domain names for multiple domains on one LB
- Monitor certificate expiration dates

## Common Issues

- symptoms: "Certificate stuck in PENDING_VALIDATION"
  diagnosis: "DNS validation CNAME records not added or incorrect."
  resolution: "Add the exact CNAME records provided by Lightsail to your DNS zone."

- symptoms: "HTTPS returns certificate error in browser"
  diagnosis: "Certificate domain doesn't match the URL or certificate not attached."
  resolution: "Verify certificate domain names. Attach certificate to load balancer."

- symptoms: "Certificate expired"
  diagnosis: "Auto-renewal failed, usually due to DNS validation records removed."
  resolution: "Ensure DNS validation records remain in place. Create new certificate if needed."

## Output Format

```yaml
root_cause: "ssl_tls — <specific_cause>"
evidence:
  - type: certificate_status
    content: "<certificate state and domains>"
  - type: dns_validation
    content: "<validation record status>"
severity: HIGH
mitigation:
  immediate: "Fix certificate validation and attachment"
  long_term: "Keep DNS validation records in place for auto-renewal"
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

- **NEVER** delete a static IP address attached to a production instance without reassignment planning
- **NEVER** modify firewall rules to allow unrestricted inbound access (0.0.0.0/0) on sensitive ports
- **NEVER** delete snapshots that serve as the only backup for production instances or databases
- **NEVER** force-stop a managed database instance without verifying active connections and transactions
- **NEVER** detach a disk from a running instance without first unmounting it from the operating system

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
