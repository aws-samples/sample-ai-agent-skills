---
title: "B1 — Place Search Issues"
description: "Diagnose place search and POI lookup problems"
status: active
severity: MEDIUM
triggers: ["search returns no results", "place not found", "SearchPlaceIndex error"]
owner: devops-agent
objective: "Identify and resolve place search issues"
context: "Place search uses provider-specific POI databases. Results vary by provider. Supports text search, position-based search, and suggestions."
---

## Phase 1 — Triage

MUST:
- Check place index: `aws location describe-place-index --index-name <name>`
- Test search: `aws location search-place-index-for-text --index-name <name> --text "Seattle"`
- Verify provider covers the search area
- Check search parameters (BiasPosition, FilterCountries)

SHOULD:
- Test with different search terms
- Check if GrabMaps used outside Southeast Asia
- Verify language settings

MAY:
- Compare results across providers
- Test with coordinates for reverse geocoding

## Phase 2 — Remediate

MUST:
- Use provider with coverage for target area
- Use appropriate search parameters
- Check for GrabMaps regional limitation

SHOULD:
- Use BiasPosition to improve local results
- Use FilterCountries to narrow results

MAY:
- Implement search result caching

## Common Issues

- symptoms: "Search returns no results"
  diagnosis: "Provider doesn't cover the area or GrabMaps used outside SE Asia."
  resolution: "Switch to Esri or HERE for global coverage."

- symptoms: "Search returns irrelevant results"
  diagnosis: "No BiasPosition set or wrong country filter."
  resolution: "Set BiasPosition near target area. Use FilterCountries."

## Output Format

```yaml
root_cause: "place_search — <specific_cause>"
evidence:
  - type: search_result
    content: "<search response>"
severity: MEDIUM
mitigation:
  immediate: "Fix provider or search parameters"
  long_term: "Implement search quality monitoring"
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
