---
title: "C1 — Schema Registry Issues"
description: "Diagnose EventBridge Schema Registry configuration and access problems"
status: active
severity: LOW
triggers:
  - "schema registry"
  - "schema not found"
  - "registry error"
  - "schema version"
owner: devops-agent
objective: "Identify and resolve Schema Registry configuration and access issues"
context: "EventBridge Schema Registry stores event schemas for code generation and validation. Registries can be AWS-managed (aws.events), discovered (discovered-schemas), or custom. Issues include schema access, version management, and code generation."
---

## Phase 1 — Triage

MUST:
- List registries: `aws schemas list-registries`
- List schemas in the registry: `aws schemas list-schemas --registry-name <registry-name>`
- Describe the schema: `aws schemas describe-schema --registry-name <registry-name> --schema-name <schema-name>`
- Check IAM permissions for schema access

SHOULD:
- Check schema versions: `aws schemas list-schema-versions --registry-name <registry-name> --schema-name <schema-name>`
- Verify the schema content matches expected event structure
- Check for schema discovery status on the event bus
- Review code generation bindings

MAY:
- Export schema for external use: `aws schemas export-schema --registry-name <registry-name> --schema-name <schema-name> --type OpenApi3`
- Check for schema policy restrictions
- Review schema tags

## Phase 2 — Remediate

MUST:
- Fix IAM permissions for schema access
- Create or update schemas to match event structures
- Verify schema registry name and schema name are correct

SHOULD:
- Use schema discovery for automatic schema creation
- Version schemas for backward compatibility
- Generate code bindings for application development

MAY:
- Implement schema governance policies
- Set up schema change notifications

## Common Issues

- symptoms: "Schema not found in registry"
  diagnosis: "Schema name is incorrect or schema is in a different registry."
  resolution: "List schemas in the registry. Check aws.events, discovered-schemas, and custom registries."

- symptoms: "Code generation fails for schema"
  diagnosis: "Schema format is invalid or unsupported for the target language."
  resolution: "Verify schema is valid OpenAPI 3.0 or JSONSchema Draft4. Check supported languages."

- symptoms: "Schema does not match actual events"
  diagnosis: "Schema was manually created and is out of date, or discovery has not captured recent changes."
  resolution: "Enable schema discovery to auto-update. Manually update the schema to match current events."

## Output Format

```yaml
root_cause: "schema_registry — <specific_cause>"
evidence:
  - type: registry
    content: "<registry and schema details>"
  - type: schema
    content: "<schema content and versions>"
  - type: permissions
    content: "<IAM permissions for schema access>"
severity: LOW
mitigation:
  immediate: "Fix schema or permissions"
  long_term: "Implement schema governance and discovery"
```


## Safety Ratings
```
safety_ratings:
  - "List registries and schemas: GREEN — read-only API call"
  - "Describe schema: GREEN — read-only API call"
  - "Check schema versions: GREEN — read-only API call"
  - "Create or update schema: GREEN — adds or updates schema definition, no runtime impact"
  - "Export schema: GREEN — read-only export operation"
  - "Delete schema version: YELLOW — removes schema version, may break code generation"
```

## Escalation Conditions
- Schema registry used by production applications for code generation
- Schema changes breaking backward compatibility
- Schema access issues affecting multiple development teams
- Registry policy restrictions blocking legitimate access

## Data Sensitivity
```
data_sensitivity:
  classification: LOW
  sensitive_fields:
    - "Schema definitions: event structure and field names"
    - "Registry names: organizational structure"
  handling: "Schema definitions may reveal data model details. Restrict access to authorized teams."
```

## Prohibited Actions
- NEVER delete a schema that is actively used for code generation in production
- NEVER modify a schema in a way that breaks backward compatibility without versioning
- NEVER delete the aws.events or discovered-schemas registries

## Phase 3 — Rollback
- If schema was updated: create a new version with the previous schema content
- If schema was deleted: recreate the schema from backup or code generation source
- If schema version was deleted: cannot be restored — recreate the version content as a new version
- If registry policy was changed: restore previous policy configuration

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
