---
title: "B2 — Firewall Configuration Issues"
description: "Diagnose Lightsail instance firewall rule problems"
status: active
severity: HIGH
triggers:
  - "firewall rules"
  - "port not open"
  - "cannot access service"
  - "connection blocked"
owner: devops-agent
objective: "Identify and resolve Lightsail firewall configuration issues"
context: "Lightsail has a built-in per-instance firewall controlling inbound traffic. Default rules allow SSH (22) and HTTP (80). Additional ports must be explicitly opened. This is NOT VPC security groups. Outbound traffic is unrestricted. IPv4 and IPv6 rules are configured separately."
---

## Phase 1 — Triage

MUST:
- Check current firewall rules: `aws lightsail get-instance-port-states --instance-name <instance-name>`
- Identify which port/protocol needs to be open
- Verify the service is actually listening on the expected port (SSH into instance)
- Check if both IPv4 and IPv6 rules are needed

SHOULD:
- Compare required ports with current firewall configuration
- Check if OS-level firewall (iptables/ufw) is also blocking
- Verify source IP restrictions if configured

MAY:
- Review application documentation for required ports
- Check if the application binds to 0.0.0.0 vs 127.0.0.1

## Phase 2 — Remediate

MUST:
- Open required ports: `aws lightsail put-instance-public-ports --instance-name <name> --port-infos fromPort=443,toPort=443,protocol=tcp`
- Note: put-instance-public-ports REPLACES all rules — include existing rules you want to keep

SHOULD:
- Open individual port without replacing: `aws lightsail open-instance-public-ports --instance-name <name> --port-info fromPort=443,toPort=443,protocol=tcp`
- Restrict source IPs where possible for security

MAY:
- Configure OS-level firewall as additional defense layer
- Document all required firewall rules

## Common Issues

- symptoms: "HTTPS not working but HTTP works"
  diagnosis: "Port 443 not open in Lightsail firewall."
  resolution: "Open port 443: aws lightsail open-instance-public-ports --port-info fromPort=443,toPort=443,protocol=tcp"

- symptoms: "Custom application port not accessible"
  diagnosis: "Custom port not added to Lightsail firewall."
  resolution: "Open the custom port in Lightsail firewall."

- symptoms: "All ports closed after firewall update"
  diagnosis: "Used put-instance-public-ports which replaces all rules."
  resolution: "Use open-instance-public-ports to add rules. Re-add missing rules."

## Output Format

```yaml
root_cause: "firewall — <specific_cause>"
evidence:
  - type: current_rules
    content: "<current firewall port states>"
  - type: required_ports
    content: "<ports needed for the application>"
severity: HIGH
mitigation:
  immediate: "Open required ports in Lightsail firewall"
  long_term: "Document required ports and use IaC for firewall management"
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
