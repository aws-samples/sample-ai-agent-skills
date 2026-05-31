---
title: "Z1 — General Lightsail Troubleshooting"
description: "Catch-all systematic investigation for Lightsail issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Lightsail issue"
  - "Lightsail problem"
  - "Lightsail error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Lightsail issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Lightsail issues that don't clearly match a specific runbook. It covers instance health, networking, databases, containers, storage, load balancers, and CDN. Use this as a starting point, then pivot to a specific runbook."
---

## Phase 1 — Triage

MUST:
- List all instances: `aws lightsail get-instances --query 'instances[*].{Name:name,State:state.name,IP:publicIpAddress}'`
- Check static IPs: `aws lightsail get-static-ips`
- Check databases: `aws lightsail get-relational-databases`
- Check container services: `aws lightsail get-container-services`
- Check load balancers: `aws lightsail get-load-balancers`
- Check distributions: `aws lightsail get-distributions`

SHOULD:
- Check instance firewall rules for affected instances
- Review recent operations and changes
- Check instance metrics for anomalies

MAY:
- Check AWS Health Dashboard for Lightsail service issues
- Review account-level limits and quotas

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Instance → A1 (creation), A2 (connectivity), A3 (performance)
  - Networking → B1 (static IP), B2 (firewall), B3 (DNS)
  - Database → C1 (managed DB), C2 (connectivity)
  - Container → D1 (service), D2 (deployment)
  - Storage → E1 (disks), E2 (snapshots)
  - Load Balancer → F1 (config), F2 (SSL/TLS)
  - CDN → G1 (distribution), G2 (caching)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Verify fixes don't introduce new problems

MAY:
- Set up monitoring and alerting
- Create documentation for common issues

## Common Issues

- symptoms: "Lightsail not working — no specific error"
  diagnosis: "Multiple possible causes across instance, network, or services."
  resolution: "Follow triage: instance state → networking → specific service."

- symptoms: "Everything was working but stopped"
  diagnosis: "Recent change or instance/service state change."
  resolution: "Check instance states. Review recent changes."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: instance_status
    content: "<instance states>"
  - type: network_config
    content: "<static IPs, firewall, DNS>"
  - type: service_status
    content: "<databases, containers, LBs, CDN>"
severity: MEDIUM
general_analysis:
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring"
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
