---
title: "A3 — Instance Performance Issues"
description: "Diagnose Lightsail instance performance degradation"
status: active
severity: MEDIUM
triggers:
  - "instance slow"
  - "high CPU"
  - "performance degradation"
  - "CPU throttling"
owner: devops-agent
objective: "Identify and resolve Lightsail instance performance issues"
context: "Lightsail smaller plans ($3.50-$10) have burstable CPU. Sustained high CPU depletes burst capacity causing throttling. Larger plans ($20+) have dedicated CPU. Performance issues also stem from insufficient memory, disk I/O, or network bandwidth exceeding plan limits."
---

## Phase 1 — Triage

MUST:
- Check CPU metrics: `aws lightsail get-instance-metric-data --instance-name <name> --metric-name CPUUtilization --period 300 --start-time <start> --end-time <end> --unit Percent --statistics Average`
- Check burst capacity: `aws lightsail get-instance-metric-data --instance-name <name> --metric-name BurstCapacityPercentage --period 300 --start-time <start> --end-time <end> --unit Percent --statistics Average`
- Check network metrics: NetworkIn, NetworkOut
- Verify instance bundle size is appropriate for workload

SHOULD:
- Check if CPU is consistently in burstable zone
- Review data transfer usage against plan allowance
- Check disk I/O if available

MAY:
- SSH into instance to check process-level resource usage (top, htop)
- Review application logs for performance-related errors

## Phase 2 — Remediate

MUST:
- If burst capacity depleted: upgrade to larger plan or reduce CPU usage
- If memory insufficient: upgrade plan (requires snapshot + new instance)
- If data transfer exceeded: monitor usage or upgrade plan

SHOULD:
- Upgrade instance: create snapshot, launch new instance with larger bundle
- Optimize application to reduce resource consumption

MAY:
- Implement caching to reduce CPU/memory pressure
- Use Lightsail CDN to offload static content delivery

## Common Issues

- symptoms: "Instance becomes very slow periodically"
  diagnosis: "Burstable CPU plan with burst capacity depleted."
  resolution: "Upgrade to $20+ plan for dedicated CPU. Or reduce sustained CPU usage."

- symptoms: "High CPU but application is idle"
  diagnosis: "Background processes consuming CPU (updates, cron jobs, malware)."
  resolution: "SSH in and check running processes. Kill unnecessary processes."

- symptoms: "Network performance degraded"
  diagnosis: "Data transfer allowance exceeded or network-intensive workload."
  resolution: "Check data transfer usage. Upgrade plan for more bandwidth."

## Output Format

```yaml
root_cause: "performance — <specific_cause>"
evidence:
  - type: cpu_metrics
    content: "<CPU utilization and burst capacity>"
  - type: instance_plan
    content: "<current bundle and resources>"
  - type: network_metrics
    content: "<network in/out and data transfer>"
severity: MEDIUM
mitigation:
  immediate: "Address the specific performance bottleneck"
  long_term: "Right-size instance plan for workload requirements"
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
