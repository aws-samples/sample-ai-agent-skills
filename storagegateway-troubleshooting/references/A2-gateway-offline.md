---
title: "A2 — Gateway Offline"
description: "Diagnose why a Storage Gateway is offline or unreachable"
status: active
severity: HIGH
triggers:
  - "gateway offline"
  - "gateway unreachable"
  - "gateway not responding"
  - "gateway disconnected"
owner: devops-agent
objective: "Identify why a Storage Gateway is offline and restore connectivity"
context: "A gateway goes offline when it loses connectivity to the AWS Storage Gateway service. Causes include network issues, VM crashes, resource exhaustion, maintenance windows, and firewall changes. An offline gateway cannot serve file shares, volumes, or tapes."
---

## Phase 1 — Triage

MUST:
- Check gateway status: `aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>`
- Verify gateway VM is running and accessible
- Test outbound connectivity on port 443 from gateway VM
- Check if gateway is in maintenance window: `aws storagegateway describe-maintenance-start-time --gateway-arn <gateway-arn>`

SHOULD:
- Check gateway VM resource utilization (CPU, memory, disk)
- Verify DNS resolution for Storage Gateway service endpoints
- Check security groups and NACLs for port 443 outbound
- Review CloudTrail for recent gateway configuration changes

MAY:
- Check gateway local console for error messages (port 2222)
- Verify NTP synchronization on gateway VM

## Phase 2 — Remediate

MUST:
- Restore network connectivity on port 443 outbound
- Restart gateway VM if unresponsive
- Verify gateway returns to RUNNING state after fix

SHOULD:
- Check that file shares/volumes/tapes resume normal operation
- Monitor gateway health metrics after recovery

MAY:
- Set up CloudWatch alarms for gateway health
- Configure redundant network paths

## Common Issues

- symptoms: "Gateway shows offline in console"
  diagnosis: "Network connectivity lost to AWS on port 443."
  resolution: "Check security groups, NACLs, route tables. Verify DNS resolution."

- symptoms: "Gateway offline during specific time window"
  diagnosis: "Maintenance window causing brief unavailability."
  resolution: "Check maintenance schedule. Reschedule if needed."

- symptoms: "Gateway VM crashed"
  diagnosis: "Resource exhaustion (memory, disk) or hypervisor issue."
  resolution: "Restart VM. Check resource allocation. Increase if undersized."

## Output Format

```yaml
root_cause: "gateway_offline — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<gateway operational state>"
  - type: network_test
    content: "<port 443 connectivity result>"
  - type: vm_health
    content: "<VM resource utilization>"
severity: HIGH
mitigation:
  immediate: "Restore gateway connectivity"
  long_term: "Set up monitoring and alerting for gateway health"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Gateway volumes contain production application data
- Tape archives may contain regulated backup data
- File shares expose S3 bucket contents to on-premises systems
- Classification: CONFIDENTIAL - production data in transit and at rest

## Prohibited Actions

- NEVER shut down or reboot gateway VMs during active data transfers
- NEVER delete volumes or tapes containing unverified backup data
- NEVER modify cache disk configuration on production gateways without maintenance window
- NEVER change iSCSI targets or NFS/SMB shares during business hours without approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

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
