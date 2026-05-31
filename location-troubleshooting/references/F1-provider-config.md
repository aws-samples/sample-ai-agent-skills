---
title: "F1 — Provider Configuration Issues"
description: "Diagnose map data provider configuration problems"
status: active
severity: MEDIUM
triggers: ["provider error", "wrong provider", "provider configuration"]
owner: devops-agent
objective: "Identify and resolve provider configuration issues"
context: "Location Service supports Esri, HERE, GrabMaps, and OpenData providers. Each has different capabilities, coverage, and pricing. Provider set at resource creation."
---

## Phase 1 — Triage

MUST:
- Check resource provider: `aws location describe-map --map-name <name>` (or place-index, route-calculator)
- Verify provider matches requirements
- Check provider capabilities for needed features
- Verify provider covers target geographic area

SHOULD:
- Compare provider capabilities
- Check provider-specific limitations
- Verify pricing tier

MAY:
- Test same operation across providers
- Review provider documentation

## Phase 2 — Remediate

MUST:
- Choose provider matching coverage and feature needs
- Create new resource with correct provider (cannot change provider)
- Use Esri or HERE for global coverage

SHOULD:
- Document provider selection rationale
- Test provider capabilities before production

MAY:
- Create resources with multiple providers for fallback

## Common Issues

- symptoms: "Feature not available with current provider"
  diagnosis: "Provider doesn't support the requested feature."
  resolution: "Switch to provider that supports the feature. Create new resource."

- symptoms: "No results in certain regions"
  diagnosis: "Provider doesn't cover the region (e.g., GrabMaps outside SE Asia)."
  resolution: "Use Esri or HERE for global coverage."

## Output Format

```yaml
root_cause: "provider_config — <specific_cause>"
evidence:
  - type: resource_config
    content: "<resource provider configuration>"
severity: MEDIUM
mitigation:
  immediate: "Create resource with correct provider"
  long_term: "Document provider capabilities and selection criteria"
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
