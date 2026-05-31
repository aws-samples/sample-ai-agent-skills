---
title: "F1 — Gateway Maintenance"
description: "Diagnose gateway maintenance and update issues"
status: active
severity: MEDIUM
triggers: ["gateway update", "maintenance window", "gateway version", "software update"]
owner: devops-agent
objective: "Resolve gateway maintenance and software update issues"
context: "Gateway software updates are automatic. Updates may cause brief connectivity interruptions. Gateway VM requires periodic maintenance."
---

## Phase 1 — Triage

MUST:
- Check gateway status and last seen time: `aws backup-gateway list-gateways --query 'Gateways[*].{Name:GatewayDisplayName,LastSeen:LastSeenTime}'`
- Get gateway details: `aws backup-gateway get-gateway --gateway-arn <gateway-arn>`
- Verify the gateway VM is healthy in vCenter (CPU, memory, disk usage)

SHOULD:
- Check if a software update was recently applied
- Verify gateway VM disk space is not exhausted

MAY:
- Review CloudTrail for gateway maintenance events
- Check ESXi host health and resource availability

## Phase 2 — Remediate

MUST:
- Ensure gateway VM has adequate disk space for updates and caching
- Verify the gateway reconnects after software updates
- Monitor gateway health after maintenance activities

SHOULD:
- Schedule maintenance windows during off-peak hours
- Ensure no backup/restore jobs are running during planned maintenance

MAY:
- Set up CloudWatch alarms for gateway health metrics
- Document the maintenance schedule and procedures

## Safety Ratings

```
safety_ratings:
  - "backup-gateway list-gateways: GREEN — read-only gateway listing"
  - "backup-gateway get-gateway: GREEN — read-only gateway details"
  - "gateway VM maintenance: YELLOW — may interrupt active operations"
  - "gateway VM disk cleanup: YELLOW — removes cached data"
```

## Escalation Conditions

- Gateway does not reconnect after a software update
- Gateway VM disk is full and cannot be expanded
- ESXi host maintenance affects the gateway VM
- Gateway requires manual intervention after an automatic update

## Data Sensitivity

- `aws backup-gateway list-gateways`: LOW — returns gateway metadata
- `aws backup-gateway get-gateway`: LOW — returns gateway configuration
- Gateway VM disk contents: MEDIUM — may contain cached backup data

## Prohibited Actions

- NEVER delete the gateway VM for maintenance — use proper shutdown procedures
- NEVER reduce gateway VM resources during maintenance
- NEVER perform maintenance during active backup or restore operations
- NEVER skip verifying gateway health after maintenance

## Phase 3 — Rollback

1. If the gateway VM was shut down for maintenance, power it back on
2. Verify the gateway reconnects to AWS after restart
3. Check that hypervisor connections are re-established
4. Verify backup schedules resume after maintenance
5. Document the maintenance activity and any issues encountered

## Common Issues

- symptoms: "Gateway went offline briefly and backup jobs failed"
  diagnosis: "Automatic software update caused a brief connectivity interruption."
  resolution: "Verify gateway is back online. Failed backup jobs will be retried on the next schedule. No action needed if gateway is healthy."

## Output Format

```yaml
root_cause: "gateway_maintenance — <specific_cause>"
evidence:
  - type: maintenance_event
    content: "<event details>"
severity: MEDIUM
mitigation:
  immediate: "Verify gateway health"
  long_term: "Set up maintenance monitoring and alerting"
```

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
  - "NEVER suggest disabling vault lock"
  - "NEVER suggest deleting backup plans without review"
  - "NEVER suggest removing cross-account backup copies"
