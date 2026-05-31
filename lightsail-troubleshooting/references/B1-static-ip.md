---
title: "B1 — Static IP Issues"
description: "Diagnose static IP allocation, attachment, and billing problems"
status: active
severity: MEDIUM
triggers:
  - "static IP"
  - "IP address changed"
  - "cannot attach IP"
  - "IP not working"
owner: devops-agent
objective: "Identify and resolve static IP configuration issues"
context: "Lightsail static IPs provide persistent public IP addresses. They are free when attached to running instances but cost $0.005/hour when unattached or on stopped instances. Maximum 5 per region. Static IPs are NOT EC2 Elastic IPs."
---

## Phase 1 — Triage

MUST:
- List static IPs: `aws lightsail get-static-ips --query 'staticIps[*].{Name:name,IP:ipAddress,Attached:isAttached,Instance:attachedTo}'`
- Check if IP is attached to correct instance
- Verify instance is running (unattached IPs on stopped instances cost money)
- Check static IP limit (5 per region)

SHOULD:
- Verify DNS records point to the correct static IP
- Check if instance was recently stopped/started without static IP

MAY:
- Review billing for unexpected static IP charges
- Check if static IP is in the correct region

## Phase 2 — Remediate

MUST:
- Attach static IP to instance: `aws lightsail attach-static-ip --static-ip-name <ip-name> --instance-name <instance-name>`
- Release unused static IPs: `aws lightsail release-static-ip --static-ip-name <ip-name>`

SHOULD:
- Allocate static IP before deploying production instances: `aws lightsail allocate-static-ip --static-ip-name <name>`
- Update DNS records after IP changes

MAY:
- Automate static IP management with scripts
- Document static IP assignments

## Common Issues

- symptoms: "Public IP changed after instance restart"
  diagnosis: "Instance does not have a static IP attached."
  resolution: "Allocate and attach a static IP to the instance."

- symptoms: "Cannot allocate more static IPs"
  diagnosis: "Reached 5 static IP limit per region."
  resolution: "Release unused static IPs or request limit increase."

- symptoms: "Unexpected charges for static IP"
  diagnosis: "Static IP not attached to a running instance."
  resolution: "Attach to running instance or release if not needed."

## Output Format

```yaml
root_cause: "static_ip — <specific_cause>"
evidence:
  - type: static_ips
    content: "<static IP list and attachment status>"
  - type: instance_state
    content: "<attached instance state>"
severity: MEDIUM
mitigation:
  immediate: "Attach or release static IPs as needed"
  long_term: "Always attach static IPs to production instances"
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
