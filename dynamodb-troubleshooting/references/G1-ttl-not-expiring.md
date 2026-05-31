---
title: "G1 — TTL Not Expiring Items"
description: "Diagnose TTL attribute format issues (epoch seconds) and understand the 48-hour deletion delay"
status: active
severity: MEDIUM
triggers:
  - "TTL not working"
  - "items not expiring"
  - "TTL delay"
  - "expired items still visible"
  - "TimeToLiveStatus"
owner: devops-agent
objective: "Identify and resolve issues with DynamoDB TTL not expiring items as expected"
context: "DynamoDB TTL deletes expired items automatically at no cost. The TTL attribute must be a Number type containing a Unix epoch timestamp in seconds (not milliseconds). Items are typically deleted within 48 hours after expiration — this is not instant. Expired items may still appear in queries and scans until actually deleted. TTL deletions appear in DynamoDB Streams as system deletes."
---

## Phase 1 — Triage

MUST:
- Check TTL configuration: `aws dynamodb describe-time-to-live --table-name <table>`
- Verify TTL attribute exists and is a Number type (not String)
- Verify the TTL value is in epoch seconds (not milliseconds): `aws dynamodb get-item --table-name <table> --key '<key-json>' --projection-expression "#ttl" --expression-attribute-names '{"#ttl":"<ttl-attribute>"}'`
- Check if TTL status is ENABLED: `aws dynamodb describe-time-to-live --table-name <table> --query 'TimeToLiveDescription.TimeToLiveStatus'`

SHOULD:
- Compare TTL values with current epoch time to verify they are in the past
- Check if the TTL attribute name matches the configured attribute exactly (case-sensitive)
- Verify items have the TTL attribute set (items without it are never expired)
- Check how long ago items should have expired (up to 48 hours is normal)

MAY:
- Scan for items with expired TTL to assess the backlog: `aws dynamodb scan --table-name <table> --filter-expression "#ttl < :now AND attribute_exists(#ttl)" --expression-attribute-names '{"#ttl":"<ttl-attribute>"}' --expression-attribute-values '{":now":{"N":"<current-epoch>"}}' --select COUNT`
- Check DynamoDB Streams for TTL deletion events (userIdentity.principalId = "dynamodb.amazonaws.com")
- Review if the table has a large backlog of expired items

## Phase 2 — Remediate

MUST:
- Ensure TTL attribute is Number type with Unix epoch in seconds
- If TTL value is in milliseconds, divide by 1000 and update items
- If TTL attribute name is wrong, update the TTL configuration: `aws dynamodb update-time-to-live --table-name <table> --time-to-live-specification Enabled=true,AttributeName=<correct-attribute>`
- Wait up to 48 hours for items to be deleted after expiration

SHOULD:
- Add FilterExpression to queries to exclude expired items while waiting for deletion
- Verify TTL is working by creating a test item with TTL set to a few minutes in the future
- Monitor DynamoDB Streams for TTL deletion events to confirm TTL is active

MAY:
- Implement a cleanup job for items that need immediate removal (don't rely on TTL for time-critical deletion)
- Use DynamoDB Streams to trigger actions when TTL deletes items
- Set TTL values with some buffer to account for the deletion delay

## Common Issues

- symptoms: "TTL attribute is set but items never expire"
  diagnosis: "TTL value is in milliseconds instead of seconds. The epoch value is interpreted as a date far in the future."
  resolution: "Convert TTL values from milliseconds to seconds (divide by 1000). Update existing items."

- symptoms: "TTL is enabled but some items don't have the TTL attribute"
  diagnosis: "Items without the TTL attribute are never expired by TTL. TTL only applies to items that have the configured attribute."
  resolution: "Add the TTL attribute to items that should expire. Use UpdateItem to set the attribute on existing items."

- symptoms: "Items expired 2 days ago but still appear in queries"
  diagnosis: "TTL deletion can take up to 48 hours. This is within normal behavior."
  resolution: "Use FilterExpression to exclude expired items: `#ttl > :now`. This is expected behavior — TTL is best-effort."

## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Fix TTL attribute format (epoch seconds) | GREEN | Data correction; no structural change |
| Update TTL configuration | GREEN | Configuration change; no data impact |
| Add FilterExpression to queries | GREEN | Application-level change; no infrastructure impact |
| Create test item with TTL | GREEN | Verification step; minimal impact |

## Escalation Conditions

- TTL misconfiguration causing millions of items to not expire (storage cost impact)
- TTL attribute in wrong format across entire table (bulk update needed)
- Expired items causing application logic errors in production
- TTL deletion backlog exceeding 48 hours on a large table

## Data Sensitivity

| Triage Command | Sensitivity | Rationale |
|---|---|---|
| `describe-time-to-live` | LOW | TTL configuration only |
| `get-item` (TTL attribute) | MEDIUM | May expose actual item data |
| `scan` (expired items count) | HIGH | Reads table data; may expose sensitive records |

## Prohibited Actions

- NEVER suggest deleting a table to fix TTL issues
- NEVER suggest disabling encryption as part of TTL fixes
- NEVER suggest removing auto-scaling without understanding traffic patterns
- NEVER suggest switching capacity mode during peak traffic hours
- NEVER rely on TTL for time-critical deletion (use explicit DeleteItem for immediate removal)

## Phase 3 — Rollback

If TTL changes cause issues:
1. If TTL was enabled on wrong attribute, update to correct attribute: `aws dynamodb update-time-to-live --table-name <table> --time-to-live-specification Enabled=true,AttributeName=<correct-attribute>`
2. If TTL is deleting items that should be retained, disable TTL: `aws dynamodb update-time-to-live --table-name <table> --time-to-live-specification Enabled=false,AttributeName=<ttl-attribute>`
3. If items were incorrectly deleted by TTL, restore from PITR backup: `aws dynamodb restore-table-to-point-in-time --source-table-name <table> --target-table-name <table-restored> --restore-date-time <timestamp>`
4. If FilterExpression was added incorrectly, revert application query code

## Output Format

```yaml
root_cause: "ttl_not_expiring — <specific_cause>"
evidence:
  - type: ttl_config
    content: "<TTL configuration>"
  - type: ttl_values
    content: "<sample TTL values and current epoch>"
  - type: item_analysis
    content: "<expired items count and age>"
severity: MEDIUM
ttl_analysis:
  ttl_enabled: "yes | no"
  ttl_attribute: "<attribute name>"
  ttl_format: "epoch_seconds | epoch_milliseconds | other"
  expired_items_count: "<count>"
  oldest_expired_item_age_hours: "<hours>"
mitigation:
  immediate: "Fix TTL attribute format and add FilterExpression to queries"
  long_term: "Implement monitoring for TTL deletion rate"
```

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
  - "NEVER suggest disabling encryption at rest"
  - "NEVER suggest deleting a table to fix issues"
  - "NEVER run full table scans on production tables"
