---
title: "E1 — Fleet Indexing Errors"
description: "Diagnose fleet indexing configuration, query failures, and stale index issues"
status: active
severity: MEDIUM
triggers: ["fleet indexing not working", "search-index returns empty", "indexing configuration error", "stale index results"]
owner: devops-agent
objective: "Resolve fleet indexing configuration and query issues"
context: "Fleet indexing enables searching things by attributes, connectivity, and shadow data. Indexing is eventually consistent. Must be enabled before dynamic groups or search queries work."
---

## Phase 1 — Triage

MUST:
- Check indexing configuration: `aws iot get-indexing-configuration`
- Test a basic search query: `aws iot search-index --query-string "thingName:*" --max-results 5`
- Verify indexing mode supports the query type (REGISTRY vs REGISTRY_AND_SHADOW vs REGISTRY_AND_CONNECTIVITY_STATUS)

SHOULD:
- Check if custom fields are configured for indexing: review thingIndexingConfiguration.customFields
- Verify managed fields include connectivity if querying connectivity status
- Test the specific failing query: `aws iot search-index --query-string "<failing-query>"`

MAY:
- Check CloudWatch metrics for indexing: `aws cloudwatch get-metric-statistics --namespace AWS/IoT --metric-name SearchIndexingFailures --start-time <time> --end-time <time> --period 3600 --statistics Sum`
- Review thing group indexing configuration if querying group membership
- Check account limits for fleet indexing

## Phase 2 — Remediate

MUST:
- Enable fleet indexing if disabled: `aws iot update-indexing-configuration --thing-indexing-configuration thingIndexingMode=REGISTRY_AND_SHADOW,thingConnectivityIndexingMode=STATUS`
- Add custom fields if querying custom attributes: include in customFields configuration
- Wait for index to build after enabling or reconfiguring (minutes to hours for large fleets)

SHOULD:
- Enable thing group indexing if querying group membership: `aws iot update-indexing-configuration --thing-group-indexing-configuration thingGroupIndexingMode=ON`
- Verify query syntax follows AWS IoT query language (not SQL)
- Test queries with small result sets before fleet-wide queries

MAY:
- Configure named shadow indexing for specific shadow names
- Set up CloudWatch alarms for indexing failures
- Document indexed fields for team reference

## Safety Ratings

```
safety_ratings:
  - "get-indexing-configuration: GREEN — read-only inspection"
  - "search-index: GREEN — read-only query"
  - "update-indexing-configuration: YELLOW — changes indexing mode, triggers reindexing, may increase costs"
```

## Escalation Conditions

- Fleet indexing consistently returns stale data after waiting for consistency window
- Indexing configuration changes cause unexpected cost increases
- Search queries time out for large fleets (100k+ things)
- Custom field indexing fails silently for specific attribute types

## Data Sensitivity

- `aws iot get-indexing-configuration`: LOW — returns indexing settings
- `aws iot search-index`: MEDIUM — returns device attributes, connectivity, and shadow data
- Indexing configuration: LOW — contains field definitions only

## Prohibited Actions

- NEVER disable fleet indexing if dynamic thing groups depend on it
- NEVER assume search results are real-time — indexing is eventually consistent
- NEVER use fleet indexing as the sole source of truth for device state
- NEVER index sensitive shadow fields that should not be queryable

## Phase 3 — Rollback

1. If indexing was misconfigured, restore previous configuration: `aws iot update-indexing-configuration --thing-indexing-configuration thingIndexingMode=<previous-mode>`
2. If custom fields were added incorrectly, remove them from configuration
3. Wait for reindexing to complete after configuration changes
4. Verify dynamic groups repopulate correctly after index rebuild
5. Confirm search queries return expected results

## Common Issues

- symptoms: "search-index returns empty results despite things existing"
  diagnosis: "Fleet indexing is disabled or indexing mode does not include the queried data (e.g., querying shadow data with REGISTRY-only mode)."
  resolution: "Enable fleet indexing with appropriate mode. Wait for initial index build to complete."

- symptoms: "Query returns results but missing recently created things"
  diagnosis: "Fleet indexing is eventually consistent. Recently created or modified things may not appear for several minutes."
  resolution: "Wait 5-10 minutes for index to update. Do not rely on immediate consistency for provisioning workflows."

## Output Format

```yaml
root_cause: "fleet_indexing — <specific_cause>"
evidence:
  - type: indexing_error
    content: "<error details>"
severity: MEDIUM
mitigation:
  immediate: "Fix indexing configuration"
  long_term: "Monitor indexing health and set up consistency expectations"
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
