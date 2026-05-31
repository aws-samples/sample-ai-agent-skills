---
title: "J2 — OpenSearch Index Rollover Issues"
description: "Diagnose and resolve index rollover failures and misconfiguration"
status: active
severity: MEDIUM
triggers:
  - "rollover"
  - "rollover failed"
  - "rollover not triggering"
  - "write alias"
  - "index rollover"
  - "data stream"
owner: devops-agent
objective: "Resolve index rollover issues and restore automated index rotation"
context: "Index rollover creates a new index when the current index meets specified conditions (age, size, doc count). Rollover requires a write alias pointing to the current index. When rollover triggers, a new index is created and the write alias moves to it. ISM policies commonly use rollover as the first action. Data streams provide built-in rollover without manual alias management. Rollover failures leave writes going to the old (potentially oversized) index."
---

## Phase 1 — Triage

MUST:
- Check rollover alias: `curl -XGET "https://<endpoint>/_cat/aliases/<alias>?v"`
- Check current write index: `curl -XGET "https://<endpoint>/_alias/<alias>?pretty"`
- Check index size and doc count: `curl -XGET "https://<endpoint>/_cat/indices/<index-pattern>?v&h=index,docs.count,store.size,creation.date.string&s=creation.date:desc"`
- Check ISM explain for rollover action: `curl -XGET "https://<endpoint>/_plugins/_ism/explain/<index>?pretty"`
- Test rollover manually: `curl -XPOST "https://<endpoint>/<alias>/_rollover?dry_run" -H 'Content-Type: application/json' -d '{"conditions":{"max_age":"1d","max_size":"50gb","max_docs":10000000}}'`

SHOULD:
- Check ISM policy rollover conditions: `curl -XGET "https://<endpoint>/_plugins/_ism/policies/<policy>?pretty"`
- Check index template for new index settings: `curl -XGET "https://<endpoint>/_index_template/<template>?pretty"`
- Check if data streams are being used: `curl -XGET "https://<endpoint>/_data_stream?pretty"`

MAY:
- Check rollover history by listing indices in the pattern
- Check for index template conflicts

## Phase 2 — Remediate

MUST:
- If no write alias: create alias with is_write_index: `curl -XPOST "https://<endpoint>/_aliases" -H 'Content-Type: application/json' -d '{"actions":[{"add":{"index":"<current-index>","alias":"<alias>","is_write_index":true}}]}'`
- If ISM rollover failed: fix conditions and retry: `curl -XPOST "https://<endpoint>/_plugins/_ism/retry/<index>" -H 'Content-Type: application/json' -d '{"state":"<state>"}'`
- If conditions never met: adjust rollover conditions to match data volume

SHOULD:
- Use index templates to ensure new rolled-over indices have correct settings and mappings
- Use data streams for time-series data (automatic rollover management)
- Set reasonable rollover conditions: max_size 50 GB, max_age 1d, or max_docs based on workload

MAY:
- Force rollover immediately: `curl -XPOST "https://<endpoint>/<alias>/_rollover" -H 'Content-Type: application/json'`
- Migrate from alias-based rollover to data streams
- Set up monitoring for rollover execution

## Common Issues

- symptoms: "Rollover not triggering despite large index"
  diagnosis: "No write alias configured, or ISM policy not attached to the index."
  resolution: "Create write alias. Attach ISM policy with rollover action."

- symptoms: "Rollover creates new index but alias doesn't move"
  diagnosis: "is_write_index not set on the alias. Old index still receives writes."
  resolution: "Set is_write_index:true on the alias for the current write index."

- symptoms: "New rolled-over index has wrong settings"
  diagnosis: "Index template not matching the rollover index name pattern."
  resolution: "Update index template to match the rollover naming pattern (e.g., my-index-*)."

## Output Format

```yaml
root_cause: "index_rollover — <specific_cause>"
evidence:
  - type: alias_config
    content: "<alias and write index configuration>"
  - type: index_state
    content: "<current index size, age, doc count>"
  - type: ism_explain
    content: "<ISM rollover action status>"
severity: MEDIUM
mitigation:
  immediate: "Fix alias configuration or ISM policy and trigger rollover"
  long_term: "Use data streams or well-configured ISM policies for automated rollover"
```


## Safety Ratings
```
safety_ratings:
  - "Check rollover alias and index state: GREEN — read-only diagnostics"
  - "Test rollover with dry_run: GREEN — read-only validation"
  - "Create write alias: YELLOW — changes write routing"
  - "Force rollover: YELLOW — creates new index and moves write alias"
  - "Update index template: GREEN — affects only new indices"
```

## Escalation Conditions
- Domain serves production search
- Rollover not triggering causing oversized indices
- Write alias misconfiguration routing writes to wrong index
- Index template conflicts affecting new rolled-over indices
- ISM rollover action failing

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index names and aliases: data routing configuration"
    - "Index sizes and doc counts: data volume information"
    - "Rollover conditions: lifecycle thresholds"
  handling: "Do not expose index naming patterns or rollover configuration externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER force rollover without verifying the index template will create the new index correctly
- NEVER remove write alias without setting it on another index first

## Phase 3 — Rollback
- If write alias was moved: move alias back to previous index
- If forced rollover created wrong index: delete the new index and restore alias to previous
- If index template was updated: revert to previous template
- If ISM rollover was retried: remove ISM policy if causing issues

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling fine-grained access control"
  - "NEVER suggest public access domains"
  - "NEVER suggest disabling encryption at rest"
