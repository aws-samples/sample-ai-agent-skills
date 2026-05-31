---
title: "D2 — Geofence Event Issues"
description: "Diagnose geofence ENTER/EXIT event problems"
status: active
severity: HIGH
triggers: ["geofence events not firing", "no ENTER event", "no EXIT event"]
owner: devops-agent
objective: "Identify and resolve geofence event issues"
context: "Geofence events (ENTER/EXIT) are sent to EventBridge when tracked devices cross boundaries. Requires tracker linked to geofence collection and EventBridge rule."
---

## Phase 1 — Triage

MUST:
- Verify tracker linked to geofence collection: `aws location list-tracker-consumers --tracker-name <name>`
- Check EventBridge rule exists for geo events
- Verify device positions are being updated
- Check geofence boundaries contain expected area

SHOULD:
- Verify position filtering isn't blocking updates
- Check EventBridge rule target configuration
- Test with manual position update inside/outside geofence

MAY:
- Check EventBridge event delivery metrics
- Review geofence evaluation latency

## Phase 2 — Remediate

MUST:
- Link tracker to geofence collection: `aws location associate-tracker-consumer --tracker-name <name> --consumer-arn <geofence-collection-arn>`
- Create EventBridge rule for `aws.geo` events
- Ensure device positions are being updated

SHOULD:
- Configure position filtering appropriately
- Test end-to-end with known positions

MAY:
- Implement geofence event monitoring

## Common Issues

- symptoms: "No geofence events despite device movement"
  diagnosis: "Tracker not linked to geofence collection."
  resolution: "Link with AssociateTrackerConsumer."

- symptoms: "Events delayed significantly"
  diagnosis: "Position filtering or evaluation latency."
  resolution: "Adjust position filtering. Account for evaluation latency."

## Output Format

```yaml
root_cause: "geofence_events — <specific_cause>"
evidence:
  - type: tracker_consumers
    content: "<tracker-geofence linkage>"
  - type: eventbridge_rules
    content: "<EventBridge rule configuration>"
severity: HIGH
mitigation:
  immediate: "Link tracker to geofence collection"
  long_term: "Implement geofence event monitoring"
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

- **NEVER** delete a geofence collection that has active tracker associations or event triggers
- **NEVER** modify tracker configurations on devices actively reporting positions in production
- **NEVER** change map data providers without verifying licensing and coverage compatibility
- **NEVER** disable position filtering on high-frequency trackers without assessing cost and rate-limit impact
- **NEVER** remove API keys or authentication for location resources serving production applications

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
