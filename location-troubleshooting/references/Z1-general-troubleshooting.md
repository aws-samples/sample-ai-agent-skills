---
title: "Z1 — General Location Service Troubleshooting"
description: "Catch-all runbook for Location Service issues"
status: active
severity: MEDIUM
triggers: ["location service error", "location not working", "geo error"]
owner: devops-agent
objective: "Systematically diagnose unspecified Location Service issues"
context: "Use this runbook when the issue doesn't clearly map to a specific runbook."
---

## Phase 1 — Triage

MUST:
- List resources: `aws location list-maps`, `aws location list-place-indexes`, `aws location list-route-calculators`, `aws location list-trackers`, `aws location list-geofence-collections`
- Check CloudTrail: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=geo.amazonaws.com --max-results 20`
- Verify IAM permissions
- Check authentication method (API key or Cognito)

SHOULD:
- Test basic operations (search, route)
- Check provider configuration
- Verify coordinate order [longitude, latitude]

MAY:
- Check service quotas
- Review service health

## Phase 2 — Remediate

MUST:
- Route to specific runbook once problem domain identified

SHOULD:
- Check coordinate order first (common mistake)
- Verify provider coverage

MAY:
- Engage AWS Support if issue persists

## Common Issues

- symptoms: "General Location Service errors"
  diagnosis: "Start with resource check, then authentication, then provider."
  resolution: "Follow systematic triage. Route to specific runbook."

- symptoms: "Everything appears in wrong location"
  diagnosis: "Coordinates swapped [lat,lng] instead of [lng,lat]."
  resolution: "Use [longitude, latitude] order throughout."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: resource_status
    content: "<resource configurations>"
severity: MEDIUM
mitigation:
  immediate: "Route to specific runbook"
  long_term: "Implement comprehensive Location Service monitoring"
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
