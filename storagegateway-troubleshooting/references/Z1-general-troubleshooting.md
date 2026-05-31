---
title: "Z1 — General Storage Gateway Troubleshooting"
description: "Catch-all systematic investigation for Storage Gateway issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers:
  - "Storage Gateway issue"
  - "gateway problem"
  - "gateway error"
  - "general troubleshooting"
owner: devops-agent
objective: "Systematically investigate Storage Gateway issues using a structured diagnostic approach"
context: "This runbook provides a general-purpose investigation framework for Storage Gateway issues that don't clearly match a specific runbook. It covers gateway health, cache status, share/volume/tape status, and systematic elimination of common causes. Use this as a starting point when the issue category is unclear, then pivot to a specific runbook once the problem area is identified."
---

## Phase 1 — Triage

MUST:
- List all gateways: `aws storagegateway list-gateways --query 'Gateways[*].{Id:GatewayId,Name:GatewayName,Type:GatewayType,State:GatewayOperationalState}'`
- Check gateway details: `aws storagegateway describe-gateway-information --gateway-arn <gateway-arn>`
- Check cache health: `aws storagegateway describe-cache --gateway-arn <gateway-arn>`
- Check bandwidth limits: `aws storagegateway describe-bandwidth-rate-limit --gateway-arn <gateway-arn>`
- List resources by type: file shares, volumes, or tapes

SHOULD:
- Check CloudTrail for recent errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=storagegateway.amazonaws.com --max-results 20`
- Check CloudWatch metrics: CacheHitPercent, UploadBufferPercentUsed, ReadBytes, WriteBytes
- Verify gateway VM health (CPU, memory, disk)

MAY:
- Check AWS Health Dashboard for service issues
- Review recent configuration changes
- Check maintenance window schedule

## Phase 2 — Remediate

MUST:
- Based on triage findings, pivot to the appropriate specific runbook:
  - Gateway issues → A1 (deployment), A2 (offline), A3 (activation)
  - File issues → B1 (file gateway), B2 (NFS/SMB), B3 (cache refresh)
  - Volume issues → C1 (volume gateway), C2 (cached vs stored)
  - Tape issues → D1 (tape gateway), D2 (virtual tapes)
  - Performance → E1 (cache), E2 (upload bandwidth)
  - Networking → F1 (connectivity), F2 (firewall ports)
  - Backup → G1 (snapshots), G2 (recovery)
  - Security → H1 (AD integration), H2 (encryption)
- Document findings and root cause

SHOULD:
- Check for multiple concurrent issues
- Verify the fix resolves the issue without side effects

MAY:
- Set up comprehensive monitoring
- Create post-incident report

## Common Issues

- symptoms: "Gateway not working — no specific error"
  diagnosis: "Multiple possible causes: connectivity, cache, permissions, resources."
  resolution: "Follow triage: gateway status → cache health → resource status → permissions."

- symptoms: "Gateway was working but stopped"
  diagnosis: "Recent change to network, permissions, or gateway configuration."
  resolution: "Check CloudTrail for changes. Compare with last known good configuration."

- symptoms: "Performance degraded gradually"
  diagnosis: "Cache filling up, bandwidth saturation, or resource exhaustion."
  resolution: "Check cache metrics, bandwidth utilization, and VM resources."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: gateway_status
    content: "<gateway operational state>"
  - type: cache_health
    content: "<cache and upload buffer metrics>"
  - type: resource_status
    content: "<file shares, volumes, or tapes status>"
severity: MEDIUM
general_analysis:
  gateway_state: "running | offline"
  cache_health: "healthy | degraded | critical"
  specific_runbook: "<recommended runbook ID>"
mitigation:
  immediate: "Follow the identified specific runbook"
  long_term: "Implement comprehensive monitoring and alerting"
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
