---
title: "B1 — VTL Mapping Template Errors"
description: "Diagnose Apache Velocity Template Language (VTL) resolver errors"
status: active
severity: HIGH
triggers:
  - "VTL error"
  - "mapping template error"
  - "Velocity template"
  - "$util error"
  - "$ctx null"
  - "resolver returns null"
owner: devops-agent
objective: "Identify and resolve VTL mapping template errors in AppSync resolvers"
context: "VTL (Apache Velocity Template Language) is the original resolver runtime for AppSync. Common issues include incorrect $ctx references, $util method misuse, JSON serialization errors, and conditional logic bugs. VTL errors often result in null responses rather than explicit errors."
---

## Phase 1 — Triage

MUST:
- Get the resolver configuration: `aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>`
- Confirm the resolver runtime is VTL (not APPSYNC_JS)
- Enable field-level logging (ALL) to see resolver execution details: `aws appsync update-graphql-api --api-id <api-id> --log-config fieldLogLevel=ALL,cloudWatchLogsRoleArn=<role-arn>,excludeVerboseContent=false`
- Check CloudWatch Logs for resolver errors: `/aws/appsync/apis/<api-id>`
- Test the resolver with a known input using the AppSync console's test resolver feature

SHOULD:
- Check for common VTL syntax errors: missing `#end`, unclosed `$util.toJson()`, incorrect `#set` usage
- Verify `$ctx.arguments` matches the GraphQL query arguments
- Check if `$ctx.source` is populated correctly for nested resolvers

MAY:
- Compare with a working VTL template for the same data source type
- Check if the response mapping template correctly handles the data source response

## Phase 2 — Remediate

MUST:
- Fix VTL syntax errors (missing #end, incorrect variable references)
- Ensure request mapping produces valid JSON for the data source
- Ensure response mapping returns valid JSON using `$util.toJson()`

SHOULD:
- Use `$util.error()` for explicit error handling instead of silent null returns
- Add `$util.validate()` for input validation
- Use `$util.appendError()` to add errors without stopping execution

MAY:
- Consider migrating to JavaScript resolvers (APPSYNC_JS) for better debugging
- Use pipeline resolvers to separate complex logic into functions

## Common Issues

- symptoms: "Resolver returns null with no error"
  diagnosis: "Response mapping template doesn't return the result. Missing $util.toJson($ctx.result)."
  resolution: "Add `$util.toJson($ctx.result)` to the response mapping template."

- symptoms: "Unable to convert to JSON" error in logs
  diagnosis: "VTL template produces invalid JSON, often from unquoted strings or missing commas."
  resolution: "Use $util.toJson() for all dynamic values. Validate JSON structure."

- symptoms: "$ctx.arguments.input is null"
  diagnosis: "Query doesn't pass the argument, or argument name doesn't match schema definition."
  resolution: "Verify the argument name in the schema matches the VTL reference."

## Output Format

```yaml
root_cause: "vtl_error — <specific_cause>"
evidence:
  - type: resolver_config
    content: "<VTL mapping template>"
  - type: cloudwatch_logs
    content: "<resolver execution logs>"
severity: HIGH
mitigation:
  immediate: "Fix the VTL template error"
  long_term: "Add resolver logging and input validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix VTL syntax errors | YELLOW | Resolver code change |
| Ensure request mapping produces valid JSON | YELLOW | Resolver code change |
| Ensure response mapping returns valid JSON | YELLOW | Resolver code change |
| Use $util.error() for error handling | YELLOW | Resolver code change |
| Add $util.validate() for input validation | YELLOW | Resolver code change |
| Consider migrating to JavaScript resolvers | YELLOW | Resolver runtime change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **VTL templates**: Resolver mapping logic — INTERNAL
- **CloudWatch Logs**: Resolver execution details — INTERNAL
- **Query arguments**: User input data — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER deploy untested VTL templates to production
- NEVER log sensitive user data in resolver templates

## Phase 3 — Rollback

1. Revert resolver to previous VTL template version
2. Update resolver: `aws appsync update-resolver --api-id <api-id> --type-name <type> --field-name <field> --request-mapping-template <previous> --response-mapping-template <previous>`
3. Verify resolver returns expected results with test queries
4. Check CloudWatch Logs for resolver errors after rollback
5. Disable verbose logging if it was enabled for debugging

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
