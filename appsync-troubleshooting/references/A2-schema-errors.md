---
title: "A2 — AppSync Schema Errors"
description: "Diagnose GraphQL schema validation and creation errors"
status: active
severity: HIGH
triggers:
  - "schema error"
  - "schema validation failed"
  - "invalid schema"
  - "SchemaCreationStatus FAILED"
  - "type definition error"
owner: devops-agent
objective: "Identify and resolve GraphQL schema errors preventing API deployment"
context: "AppSync validates the GraphQL schema during creation. Errors include invalid type definitions, missing required types (Query), duplicate type names, invalid directives, and circular references. Schema creation is asynchronous — check status after starting."
---

## Phase 1 — Triage

MUST:
- Check schema creation status: `aws appsync get-schema-creation-status --api-id <api-id>`
- Get the detailed error from the status response (details field contains specific errors)
- Review the schema for basic GraphQL validity (Query type must exist)
- Check for duplicate type definitions
- Verify all referenced types are defined

SHOULD:
- Validate the schema locally using a GraphQL linter before uploading
- Check for invalid AppSync directives (@aws_auth, @aws_subscribe, @model)
- Verify subscription types reference valid mutation fields

MAY:
- Compare with a previously working schema version
- Check if merged API source schemas have conflicts

## Phase 2 — Remediate

MUST:
- Fix the specific schema error reported in the creation status details
- Ensure a `type Query` root type exists
- Resolve duplicate type names
- Ensure all referenced types and input types are defined

SHOULD:
- Use schema validation tools (graphql-inspector, eslint-plugin-graphql) in CI/CD
- Keep schema modular using schema stitching or AppSync merged APIs

MAY:
- Enable schema introspection for debugging: `aws appsync get-introspection-schema --api-id <api-id> --format SDL`

## Common Issues

- symptoms: "SchemaCreationStatus: FAILED with 'Query type must be provided'"
  diagnosis: "Schema is missing the root Query type."
  resolution: "Add a `type Query { ... }` definition to the schema."

- symptoms: "Duplicate type definition for 'User'"
  diagnosis: "The same type name is defined multiple times in the schema."
  resolution: "Remove duplicate definitions or use extend type syntax."

- symptoms: "@aws_subscribe directive references non-existent mutation"
  diagnosis: "Subscription field references a mutation that doesn't exist in the schema."
  resolution: "Ensure the mutation name in @aws_subscribe matches an actual mutation field."

## Output Format

```yaml
root_cause: "schema_error — <specific_cause>"
evidence:
  - type: schema_creation_status
    content: "<error details from get-schema-creation-status>"
  - type: schema_definition
    content: "<problematic schema section>"
severity: HIGH
mitigation:
  immediate: "Fix the schema error and re-upload"
  long_term: "Add schema validation to CI/CD pipeline"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix specific schema error | YELLOW | Schema change |
| Ensure Query root type exists | YELLOW | Schema change |
| Resolve duplicate type names | YELLOW | Schema change |
| Ensure all referenced types defined | YELLOW | Schema change |
| Use schema validation tools in CI/CD | GREEN | Tooling improvement |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Schema definition**: GraphQL type system — INTERNAL
- **Schema creation status**: Error details — INTERNAL
- **API introspection**: Full schema exposure — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER deploy schema changes to production without testing
- NEVER remove types that existing resolvers depend on

## Phase 3 — Rollback

1. Re-upload the previous working schema: `aws appsync start-schema-creation --api-id <api-id> --definition <previous-schema>`
2. Wait for schema creation to complete: `aws appsync get-schema-creation-status --api-id <api-id>`
3. Verify all resolvers are still attached and functional
4. Test critical queries and mutations after schema rollback
5. Verify subscription types still reference valid mutations

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
  - "NEVER suggest removing authentication from APIs"
  - "NEVER suggest API_KEY auth for production"
  - "NEVER suggest disabling logging"
