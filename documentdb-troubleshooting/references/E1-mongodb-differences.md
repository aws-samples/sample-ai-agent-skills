---
title: "E1 — MongoDB Compatibility Differences"
description: "Diagnose issues caused by differences between DocumentDB and MongoDB"
status: active
severity: MEDIUM
triggers:
  - "mongodb compatibility"
  - "unsupported operation"
  - "not supported"
  - "mongodb feature"
  - "migration from mongodb"
owner: devops-agent
objective: "Identify and resolve issues caused by MongoDB feature differences in DocumentDB"
context: "DocumentDB is API-compatible with MongoDB 3.6/4.0/5.0 but does not support all features. Common differences include aggregation pipeline stages, index types, capped collections, certain commands, and storage engine behaviors."
---

## Phase 1 — Triage

MUST:
- Identify the specific unsupported operation or feature from error messages
- Check DocumentDB engine version: `aws docdb describe-db-clusters --db-cluster-identifier <cluster-id>`
- Verify the operation is supported in the cluster's engine version
- Review DocumentDB supported MongoDB APIs documentation

SHOULD:
- Check if the feature was added in a newer DocumentDB engine version
- Identify workarounds for unsupported features
- Review application code for MongoDB-specific assumptions
- Check driver compatibility with DocumentDB

MAY:
- Run the DocumentDB compatibility tool against the application
- Compare MongoDB and DocumentDB behavior for edge cases
- Review migration assessment documentation

## Phase 2 — Remediate

MUST:
- Replace unsupported operations with DocumentDB-compatible alternatives
- Upgrade engine version if the feature is available in a newer version
- Update application code to handle DocumentDB-specific behaviors

SHOULD:
- Document all MongoDB-to-DocumentDB differences affecting the application
- Test thoroughly after making compatibility changes
- Use DocumentDB-specific documentation, not MongoDB documentation

MAY:
- Consider upgrading to DocumentDB 5.0 for broader MongoDB compatibility
- Implement application-level workarounds for unsupported features

## Common Issues

- symptoms: "Unsupported command: $graphLookup"
  diagnosis: "DocumentDB has limited $graphLookup support depending on engine version."
  resolution: "Check engine version support. Implement graph traversal in application code if unsupported."

- symptoms: "Capped collection not supported"
  diagnosis: "DocumentDB does not support capped collections."
  resolution: "Use TTL indexes for automatic document expiration. Implement application-level size management."

- symptoms: "Change stream behavior differs from MongoDB"
  diagnosis: "DocumentDB change streams have 24h default retention and different resume behavior."
  resolution: "Implement resume token management. Set appropriate retention via change_stream_log_retention_duration."

## Output Format

```yaml
root_cause: "mongodb_difference — <specific_cause>"
evidence:
  - type: error
    content: "<unsupported operation error>"
  - type: engine_version
    content: "<DocumentDB engine version and supported features>"
  - type: documentation
    content: "<DocumentDB vs MongoDB feature comparison>"
severity: MEDIUM
mitigation:
  immediate: "Replace unsupported operation with compatible alternative"
  long_term: "Document all compatibility differences and test thoroughly"
```


---

## Safety Ratings

| Dimension | Rating |
|-----------|--------|
| Data Sensitivity | LOW — Compatibility analysis is operational |
| Blast Radius | MEDIUM — Unsupported operations cause application errors |
| Rollback Complexity | MEDIUM — Code changes may be needed to replace unsupported operations |
| Compliance Scope | LOW — Compatibility is an operational concern |

## Escalation Conditions

- Escalate to application developers if unsupported operations are discovered
- Escalate to architecture team if migration from MongoDB requires significant changes

## Data Sensitivity

- **Application code** may be exposed during compatibility analysis
- **Error messages** may contain query details

## PROHIBITED Actions

- Do NOT assume MongoDB documentation applies to DocumentDB — use DocumentDB docs
- Do NOT deploy applications to DocumentDB without compatibility testing
- Do NOT use unsupported features in production without workarounds

## Phase 3 — Rollback

- Revert application code to use DocumentDB-compatible operations
- If engine version was upgraded, note that downgrades are not supported — restore from snapshot
- Document all compatibility differences for the application

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
  - "NEVER suggest disabling TLS"
  - "NEVER suggest making clusters publicly accessible"
  - "NEVER suggest disabling audit logging"
