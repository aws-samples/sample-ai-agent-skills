---
title: "B3 — DNS and Domain Issues"
description: "Diagnose Lightsail DNS zone and domain resolution problems"
status: active
severity: MEDIUM
triggers:
  - "DNS not resolving"
  - "domain not working"
  - "DNS zone error"
  - "domain configuration"
owner: devops-agent
objective: "Identify and resolve DNS and domain configuration issues in Lightsail"
context: "Lightsail provides free DNS zones with support for A, AAAA, CNAME, MX, NS, SRV, and TXT records. DNS zones do NOT support alias records, weighted routing, or health checks. For advanced DNS, use Route 53. Domain registrar nameservers must point to Lightsail DNS nameservers."
---

## Phase 1 — Triage

MUST:
- Check DNS zone: `aws lightsail get-domains --query 'domains[*].{Name:name,Entries:domainEntries}'`
- Verify nameservers at registrar point to Lightsail DNS nameservers
- Check DNS record configuration for the domain
- Test DNS resolution: `nslookup <domain>` or `dig <domain>`

SHOULD:
- Verify A record points to correct static IP
- Check for conflicting CNAME and A records
- Verify TTL values are appropriate

MAY:
- Check DNS propagation status using external tools
- Review if Route 53 would be more appropriate for the use case

## Phase 2 — Remediate

MUST:
- Create DNS zone if missing: `aws lightsail create-domain --domain-name <domain>`
- Add/update DNS records: `aws lightsail create-domain-entry --domain-name <domain> --domain-entry name=<subdomain>,type=A,target=<ip>`
- Update registrar nameservers to Lightsail DNS nameservers

SHOULD:
- Use appropriate TTL values (300-3600 seconds)
- Configure both A and AAAA records if IPv6 is needed

MAY:
- Migrate to Route 53 for advanced DNS features
- Set up monitoring for DNS resolution

## Common Issues

- symptoms: "Domain not resolving to Lightsail instance"
  diagnosis: "Registrar nameservers not pointing to Lightsail DNS."
  resolution: "Update registrar nameservers to Lightsail DNS nameservers."

- symptoms: "Subdomain not working"
  diagnosis: "Missing DNS record for the subdomain."
  resolution: "Add A or CNAME record for the subdomain in Lightsail DNS zone."

- symptoms: "DNS changes not taking effect"
  diagnosis: "DNS propagation delay or cached old records."
  resolution: "Wait for TTL to expire. Check with dig +trace for propagation status."

## Output Format

```yaml
root_cause: "dns_domain — <specific_cause>"
evidence:
  - type: dns_zone
    content: "<DNS zone configuration>"
  - type: nameservers
    content: "<registrar nameserver configuration>"
  - type: resolution_test
    content: "<DNS resolution result>"
severity: MEDIUM
mitigation:
  immediate: "Fix DNS records and nameserver configuration"
  long_term: "Document DNS configuration and monitor resolution"
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
