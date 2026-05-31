---
title: "A2 — Instance Connectivity Issues"
description: "Diagnose SSH, RDP, and network connectivity problems with Lightsail instances"
status: active
severity: HIGH
triggers:
  - "cannot SSH"
  - "connection refused"
  - "instance unreachable"
  - "cannot connect to instance"
owner: devops-agent
objective: "Identify and resolve connectivity issues with Lightsail instances"
context: "Connectivity issues can stem from Lightsail firewall rules, instance state, SSH key problems, static IP configuration, or instance-level networking. The Lightsail firewall controls inbound access and is separate from VPC security groups."
---

## Phase 1 — Triage

MUST:
- Check instance state: `aws lightsail get-instance-state --instance-name <instance-name>`
- Check firewall rules: `aws lightsail get-instance-port-states --instance-name <instance-name>`
- Verify instance has a public IP: `aws lightsail get-instance --instance-name <name> --query 'instance.publicIpAddress'`
- Check if SSH key matches: `aws lightsail get-instance --instance-name <name> --query 'instance.sshKeyName'`

SHOULD:
- Test connectivity from browser-based SSH in Lightsail console
- Check if static IP is attached and correct
- Verify instance is not in a stopped or error state

MAY:
- Check instance system log for boot errors
- Verify OS-level firewall (iptables/ufw) is not blocking

## Phase 2 — Remediate

MUST:
- Ensure instance is in running state
- Open required ports in Lightsail firewall: `aws lightsail put-instance-public-ports --instance-name <name> --port-infos fromPort=22,toPort=22,protocol=tcp`
- Verify correct SSH key is being used

SHOULD:
- Attach a static IP if public IP changed after stop/start
- Use browser-based SSH as a fallback for troubleshooting

MAY:
- Check OS-level firewall via Lightsail browser SSH
- Reboot instance if unresponsive: `aws lightsail reboot-instance --instance-name <name>`

## Common Issues

- symptoms: "Connection timed out on SSH"
  diagnosis: "Port 22 not open in Lightsail firewall or instance stopped."
  resolution: "Check firewall rules. Ensure instance is running."

- symptoms: "Permission denied (publickey)"
  diagnosis: "Wrong SSH key or key not matching instance."
  resolution: "Use the correct private key matching the instance's key pair."

- symptoms: "Public IP changed after restart"
  diagnosis: "Instance without static IP gets new public IP on stop/start."
  resolution: "Attach a static IP to maintain consistent public IP."

## Output Format

```yaml
root_cause: "connectivity — <specific_cause>"
evidence:
  - type: instance_state
    content: "<instance running state>"
  - type: firewall_rules
    content: "<open ports>"
  - type: ip_config
    content: "<public IP and static IP status>"
severity: HIGH
mitigation:
  immediate: "Fix firewall rules or SSH key configuration"
  long_term: "Attach static IP and document firewall requirements"
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
