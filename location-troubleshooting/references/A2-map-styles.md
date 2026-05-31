---
title: "A2 — Map Style Issues"
description: "Diagnose map style configuration problems"
status: active
severity: LOW
triggers: ["wrong map style", "style not available", "map appearance"]
owner: devops-agent
objective: "Identify and resolve map style issues"
context: "Map styles are provider-specific. Esri, HERE, and OpenData offer different styles (street, satellite, navigation). Style must be specified at map creation."
---

## Phase 1 — Triage

MUST:
- Check map configuration: `aws location describe-map --map-name <name>`
- Verify style name is valid for the provider
- Check provider supports desired style

SHOULD:
- List available styles for the provider
- Compare styles across providers

MAY:
- Test different styles for visual comparison

## Phase 2 — Remediate

MUST:
- Use valid style name for the chosen provider
- Create new map resource for different style (cannot change style)

SHOULD:
- Choose style matching use case (navigation, satellite, etc.)

MAY:
- Create multiple map resources for style switching

## Common Issues

- symptoms: "Map shows wrong style"
  diagnosis: "Style name doesn't match expected appearance."
  resolution: "Check provider's available styles. Create new map with correct style."

- symptoms: "Satellite view not available"
  diagnosis: "Provider doesn't offer satellite style."
  resolution: "Use Esri for satellite imagery. Check provider capabilities."

## Output Format

```yaml
root_cause: "map_styles — <specific_cause>"
evidence:
  - type: map_config
    content: "<map style configuration>"
severity: LOW
mitigation:
  immediate: "Create map with correct style"
  long_term: "Document available styles per provider"
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
