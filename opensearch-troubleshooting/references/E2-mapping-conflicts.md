---
title: "E2 — OpenSearch Mapping Conflicts"
description: "Diagnose and resolve field mapping conflicts and type mismatches"
status: active
severity: MEDIUM
triggers:
  - "mapping conflict"
  - "mapper_parsing_exception"
  - "field type"
  - "type mismatch"
  - "mapping explosion"
owner: devops-agent
objective: "Resolve mapping conflicts and prevent future mapping issues"
context: "OpenSearch mappings define field types for an index. Once a field is mapped to a type, it cannot be changed without reindexing. Dynamic mapping automatically creates mappings for new fields, which can lead to type conflicts when different documents have different types for the same field name. Mapping explosion occurs when too many unique fields are created (default limit: 1000 fields per index)."
---

## Phase 1 — Triage

MUST:
- Check current mapping: `curl -XGET "https://<endpoint>/<index>/_mapping?pretty"`
- Check field count: `curl -XGET "https://<endpoint>/<index>/_mapping?pretty" | python3 -c "import sys,json; m=json.load(sys.stdin); print(len(str(m)))"`
- Check field limit setting: `curl -XGET "https://<endpoint>/<index>/_settings?pretty&filter_path=*.settings.index.mapping.total_fields.limit"`
- Test problematic document: `curl -XPOST "https://<endpoint>/<index>/_doc?dry_run" -H 'Content-Type: application/json' -d '<document>'`
- Check dynamic mapping setting: `curl -XGET "https://<endpoint>/<index>/_mapping?pretty&filter_path=*.mappings.dynamic"`

SHOULD:
- Check index templates for mapping definitions: `curl -XGET "https://<endpoint>/_index_template?pretty"`
- Compare mapping across indices in the same pattern (e.g., daily indices)
- Check for conflicting field types across indices behind an alias

MAY:
- Check mapping stats: `curl -XGET "https://<endpoint>/<index>/_stats/fielddata?pretty"`
- Review ingest pipeline for field transformations

## Phase 2 — Remediate

MUST:
- If type mismatch: reindex with correct mapping: `curl -XPOST "https://<endpoint>/_reindex" -H 'Content-Type: application/json' -d '{"source":{"index":"<old-index>"},"dest":{"index":"<new-index>"}}'`
- If mapping explosion: increase limit or fix data pipeline: `curl -XPUT "https://<endpoint>/<index>/_settings" -H 'Content-Type: application/json' -d '{"index.mapping.total_fields.limit": 2000}'`
- Create explicit mappings in index templates to prevent dynamic mapping issues

SHOULD:
- Set dynamic mapping to "strict" for production indices: `curl -XPUT "https://<endpoint>/<index>/_mapping" -H 'Content-Type: application/json' -d '{"dynamic":"strict"}'`
- Use dynamic templates to control type inference for new fields
- Standardize field names and types across data sources

MAY:
- Use coerce setting to automatically convert compatible types
- Implement ingest pipelines to normalize field types before indexing
- Use field aliases for backward compatibility

## Common Issues

- symptoms: "mapper_parsing_exception: failed to parse field [timestamp]"
  diagnosis: "Document sends string for a field mapped as date, or vice versa."
  resolution: "Fix data pipeline to send correct type. Or reindex with updated mapping."

- symptoms: "Limit of total fields [1000] has been exceeded"
  diagnosis: "Dynamic mapping created too many fields from unstructured data."
  resolution: "Increase limit, disable dynamic mapping, or flatten data structure."

- symptoms: "Different field types across daily indices"
  diagnosis: "No index template defining explicit mappings. Dynamic mapping inferred differently."
  resolution: "Create index template with explicit mappings for all known fields."

## Output Format

```yaml
root_cause: "mapping_conflict — <specific_cause>"
evidence:
  - type: mapping
    content: "<current index mapping>"
  - type: error
    content: "<mapper_parsing_exception details>"
  - type: field_count
    content: "<total field count and limit>"
severity: MEDIUM
mitigation:
  immediate: "Fix document type or reindex with correct mapping"
  long_term: "Use explicit mappings in templates, set dynamic to strict"
```


## Safety Ratings
```
safety_ratings:
  - "Check mapping and field count: GREEN — read-only diagnostics"
  - "Increase field limit: YELLOW — allows more fields, increases complexity"
  - "Set dynamic mapping to strict: YELLOW — rejects documents with unknown fields"
  - "Reindex with correct mapping: YELLOW — resource-intensive, creates new index"
  - "Create index templates: GREEN — affects only new indices"
```

## Escalation Conditions
- Domain serves production search
- Mapping conflicts blocking data ingestion
- Mapping explosion from unstructured data
- Fix requires reindex of large production indices
- Cross-index mapping inconsistencies

## Data Sensitivity
```
data_sensitivity:
  classification: HIGH
  sensitive_fields:
    - "Index mappings: data schema and field names"
    - "Document samples: business data"
  handling: "Mappings reveal data structure. Do not expose field names or document samples externally."
```

## Prohibited Actions
- NEVER suggest deleting indices without backup
- NEVER suggest reducing node count below minimum for cluster health
- NEVER change field types on existing indices (requires reindex)
- NEVER set dynamic to false without understanding that unmapped fields are silently dropped

## Phase 3 — Rollback
- If dynamic was set to strict: change back to true or runtime
- If field limit was increased: reduce back to previous value
- If reindex was performed: delete new index and keep original if reindex was incorrect
- If index template was updated: revert to previous template definition

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
