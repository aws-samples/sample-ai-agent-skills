---
title: "D2 — Reserved Pricing Issues"
description: "Diagnose reserved queue pricing and capacity configuration"
status: active
severity: MEDIUM
triggers:
  - "reserved pricing"
  - "RTS"
  - "reserved transcode"
  - "commitment"
owner: devops-agent
objective: "Resolve reserved queue pricing, capacity, and commitment issues"
context: "Reserved queues use Reserved Transcode Slots (RTS) with 12-month commitments. Each RTS processes one job at a time. RTS count is fixed and does not auto-scale. Reserved pricing is cheaper per minute than on-demand."
---

## Phase 1 — Triage

MUST:
- Check reserved queue details: `aws mediaconvert get-queue --name <queue-name> --endpoint-url <endpoint>`
- Verify RTS count and commitment status
- Check queue utilization: count PROGRESSING jobs vs RTS count
- Review pricing plan details

SHOULD:
- Analyze historical job volume to validate RTS sizing
- Check for underutilized RTS (paying for unused capacity)
- Compare reserved vs on-demand cost for the workload

MAY:
- Review commitment expiration dates
- Check for auto-renewal settings

## Phase 2 — Remediate

MUST:
- If RTS insufficient: purchase additional RTS or use on-demand for overflow
- If RTS underutilized: consider reducing at next commitment renewal
- If commitment expiring: plan renewal or migration to on-demand

SHOULD:
- Right-size RTS based on average and peak job volume
- Use on-demand queues for burst capacity beyond RTS
- Monitor RTS utilization with CloudWatch

MAY:
- Implement cost optimization analysis for reserved vs on-demand
- Set up alerts for RTS utilization thresholds

## Common Issues

- symptoms: "Reserved queue jobs waiting despite paying for RTS"
  diagnosis: "All RTS occupied by long-running jobs."
  resolution: "Wait for current jobs or submit overflow to on-demand queue."

- symptoms: "Reserved queue costs higher than expected"
  diagnosis: "RTS over-provisioned for actual job volume."
  resolution: "Analyze utilization. Reduce RTS at next renewal if consistently underutilized."

## Output Format

```yaml
root_cause: "reserved_pricing — <specific_cause>"
evidence:
  - type: rts_config
    content: "<RTS count and utilization>"
  - type: cost_analysis
    content: "<reserved vs on-demand comparison>"
severity: MEDIUM
mitigation:
  immediate: "Adjust job routing between reserved and on-demand"
  long_term: "Right-size RTS based on utilization analysis"
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

- **NEVER** delete or modify a production job queue while jobs are actively processing
- **NEVER** change IAM roles on job templates without verifying S3 input/output bucket access
- **NEVER** disable DRM or content protection settings on outputs destined for licensed content
- **NEVER** cancel reserved transcoding capacity without assessing active workload commitments
- **NEVER** modify output presets for compliance-regulated content without re-validation

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
