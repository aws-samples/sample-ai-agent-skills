---
title: "F2 — Provider Coverage Issues"
description: "Diagnose provider geographic coverage problems"
status: active
severity: MEDIUM
triggers: ["no coverage", "region not supported", "GrabMaps limitation"]
owner: devops-agent
objective: "Identify and resolve provider coverage issues"
context: "Provider coverage varies: Esri and HERE have global coverage. GrabMaps covers Southeast Asia only. OpenData has varying coverage. Coverage affects search, routing, and map quality."
---

## Phase 1 — Triage

MUST:
- Identify current provider for the resource
- Check if target region is covered by provider
- Test with known locations in target region
- Verify GrabMaps is not used outside Southeast Asia

SHOULD:
- Compare coverage across providers for target region
- Check data quality in target region
- Test place search and routing in target area

MAY:
- Review provider coverage documentation
- Test edge cases at coverage boundaries

## Phase 2 — Remediate

MUST:
- Use provider with coverage for target region
- Use Esri or HERE for global coverage
- Use GrabMaps only for Southeast Asia

SHOULD:
- Test coverage before production deployment
- Document coverage requirements

MAY:
- Implement multi-provider fallback strategy

## Common Issues

- symptoms: "No search results in target country"
  diagnosis: "GrabMaps used outside Southeast Asia."
  resolution: "Switch to Esri or HERE for global coverage."

- symptoms: "Poor data quality in specific region"
  diagnosis: "Provider has limited data for that region."
  resolution: "Try different provider. Compare data quality across providers."

## Output Format

```yaml
root_cause: "provider_coverage — <specific_cause>"
evidence:
  - type: coverage_check
    content: "<provider coverage for target region>"
severity: MEDIUM
mitigation:
  immediate: "Switch to provider with coverage"
  long_term: "Implement provider coverage validation"
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
