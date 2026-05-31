---
title: "A1 — Map Rendering Issues"
description: "Diagnose map rendering and tile loading problems"
status: active
severity: HIGH
triggers: ["map not rendering", "tiles not loading", "blank map", "map error"]
owner: devops-agent
objective: "Identify and resolve map rendering issues"
context: "Maps use tile-based rendering from configured provider. Requires proper authentication (API key or Cognito), CORS configuration for web apps, and valid map resource."
---

## Phase 1 — Triage

MUST:
- Check map resource: `aws location describe-map --map-name <name>`
- Verify map resource exists and is configured
- Check authentication method (API key or Cognito)
- Verify CORS configuration for web applications

SHOULD:
- Test map tile URL directly
- Check browser console for CORS or authentication errors
- Verify API key is valid and not expired

MAY:
- Check map provider status
- Test with different map styles

## Phase 2 — Remediate

MUST:
- Create map resource with desired provider and style
- Configure authentication (API key or Cognito)
- Set up CORS for web applications

SHOULD:
- Use HTTPS for all map tile requests
- Include proper authentication headers/parameters

MAY:
- Implement map tile caching

## Common Issues

- symptoms: "Map shows blank tiles"
  diagnosis: "Authentication failed or CORS blocking requests."
  resolution: "Check API key validity. Configure CORS. Check browser console."

- symptoms: "Map tiles load slowly"
  diagnosis: "Network latency or no caching."
  resolution: "Use CDN caching. Check network connectivity."

## Output Format

```yaml
root_cause: "map_rendering — <specific_cause>"
evidence:
  - type: map_config
    content: "<map resource configuration>"
severity: HIGH
mitigation:
  immediate: "Fix authentication or CORS"
  long_term: "Implement map tile caching and monitoring"
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
