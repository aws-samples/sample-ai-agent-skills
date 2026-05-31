---
title: "F1 — Fleet Indexing Integration"
description: "Diagnose fleet indexing integration issues with Device Defender"
status: active
severity: MEDIUM
triggers: ["fleet indexing not showing violations", "thing indexing error", "aggregation query failed", "violation data not indexed"]
owner: devops-agent
objective: "Resolve fleet indexing integration issues including indexing configuration, violation data availability, and aggregation queries"
context: "Fleet indexing can include Device Defender violation data, enabling aggregation queries across the fleet. Requires fleet indexing to be enabled with Device Defender as an indexing source."
---

## Phase 1 — Triage

MUST:
- Check fleet indexing configuration: `aws iot describe-index --index-name AWS_Things`
- Verify indexing status: `aws iot get-indexing-configuration`
- Check if Device Defender violations are included in indexing sources

SHOULD:
- Test aggregation query: `aws iot get-statistics --index-name AWS_Things --query-string "defender:*"`
- Verify things are indexed: `aws iot search-index --index-name AWS_Things --query-string "thingName:<thing-name>"`
- Check indexing backlog status

MAY:
- Review fleet metrics dashboard for violation aggregation
- Check CloudWatch for indexing errors

## Phase 2 — Remediate

MUST:
- Enable fleet indexing if not enabled: `aws iot update-indexing-configuration --thing-indexing-configuration thingIndexingMode=REGISTRY_AND_SHADOW`
- Add Device Defender violations as indexing source
- Wait for indexing to complete (can take minutes to hours for large fleets)

SHOULD:
- Verify indexing configuration includes all required data sources
- Test aggregation queries after indexing is complete

MAY:
- Set up CloudWatch dashboards using fleet indexing aggregation data
- Configure scheduled queries for fleet security posture reporting

## Safety Ratings

```
safety_ratings:
  - "iot describe-index: GREEN — read-only index status"
  - "iot get-indexing-configuration: GREEN — read-only configuration check"
  - "iot update-indexing-configuration: YELLOW — modifies indexing, may increase costs"
  - "iot search-index: GREEN — read-only search query"
```

## Escalation Conditions

- Fleet indexing fails to include Device Defender data after configuration
- Indexing backlog grows continuously indicating capacity issues
- Aggregation queries timeout on large fleets
- Indexing costs exceed budget expectations

## Data Sensitivity

- `aws iot describe-index`: LOW — returns index metadata
- `aws iot get-indexing-configuration`: LOW — returns indexing configuration
- `aws iot search-index`: MEDIUM — returns thing details and violation data
- `aggregation query results`: MEDIUM — reveal fleet-wide security posture

## Prohibited Actions

- NEVER disable fleet indexing without understanding impact on monitoring dashboards
- NEVER run expensive aggregation queries in tight loops (rate limiting)
- NEVER expose fleet indexing query results containing device details publicly
- NEVER modify indexing configuration during active incident investigation

## Phase 3 — Rollback

1. If indexing configuration was changed incorrectly, restore previous settings
2. If indexing was disabled, re-enable and wait for re-indexing
3. Verify aggregation queries return expected results after fix
4. Check that monitoring dashboards are functioning correctly
5. Document the indexing configuration and query patterns

## Common Issues

- symptoms: "Fleet indexing query returns no Device Defender data"
  diagnosis: "Device Defender violations not configured as an indexing source."
  resolution: "Update indexing configuration to include Device Defender violation data."

## Output Format

```yaml
root_cause: "fleet_indexing — <specific_cause>"
evidence:
  - type: indexing_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix indexing configuration"
  long_term: "Monitor indexing health and set up dashboards"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
