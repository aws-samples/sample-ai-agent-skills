---
title: "B2 — JavaScript Resolver Errors"
description: "Diagnose APPSYNC_JS runtime resolver errors"
status: active
severity: HIGH
triggers:
  - "JavaScript resolver error"
  - "APPSYNC_JS error"
  - "JS resolver"
  - "runtime error"
  - "util.error"
owner: devops-agent
objective: "Identify and resolve JavaScript resolver runtime errors in AppSync"
context: "APPSYNC_JS is the newer resolver runtime supporting JavaScript. It uses ctx and util (without $) and supports a subset of JavaScript. Common issues include unsupported JS features, incorrect ctx references, and async misunderstandings (APPSYNC_JS is synchronous)."
---

## Phase 1 — Triage

MUST:
- Get the resolver configuration: `aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>`
- Confirm the resolver runtime is APPSYNC_JS (runtime.name = "APPSYNC_JS")
- Enable field-level logging: `aws appsync update-graphql-api --api-id <api-id> --log-config fieldLogLevel=ALL,cloudWatchLogsRoleArn=<role-arn>,excludeVerboseContent=false`
- Check CloudWatch Logs for resolver errors: `/aws/appsync/apis/<api-id>`
- Evaluate the resolver code using the evaluate API: `aws appsync evaluate-code --runtime name=APPSYNC_JS,runtimeVersion=1.0.0 --code <code> --context <context-json> --function request`

SHOULD:
- Check for unsupported JavaScript features (no async/await, no require/import, no setTimeout)
- Verify the function exports `request` and `response` handlers
- Check that `ctx.args` (not `ctx.arguments`) is used correctly

MAY:
- Use the AppSync console code editor with built-in validation
- Test with the evaluate-code API before deploying

## Phase 2 — Remediate

MUST:
- Fix JavaScript syntax errors and unsupported feature usage
- Ensure `request` and `response` functions are exported
- Return valid request objects matching the data source type

SHOULD:
- Use `util.error()` for explicit error handling
- Use `util.appendError()` for partial error responses
- Validate inputs with `util.validate()`

MAY:
- Use TypeScript with the AppSync utilities package for type safety during development
- Migrate complex VTL templates to JavaScript for better maintainability

## Common Issues

- symptoms: "Unexpected token" or "SyntaxError" in resolver evaluation
  diagnosis: "Using unsupported JavaScript features (async/await, optional chaining, etc.)."
  resolution: "Use only supported APPSYNC_JS features. Check the supported features documentation."

- symptoms: "request function must return a value"
  diagnosis: "The request handler doesn't return a valid data source request object."
  resolution: "Return a valid request object (e.g., { operation: 'GetItem', key: util.dynamodb.toMapValues({id: ctx.args.id}) })."

- symptoms: "Cannot read property of undefined"
  diagnosis: "Accessing a nested property on ctx that doesn't exist (e.g., ctx.args.input.field when input is null)."
  resolution: "Add null checks before accessing nested properties."

## Output Format

```yaml
root_cause: "javascript_resolver_error — <specific_cause>"
evidence:
  - type: resolver_code
    content: "<JavaScript resolver code>"
  - type: evaluate_result
    content: "<evaluate-code API output>"
severity: HIGH
mitigation:
  immediate: "Fix the JavaScript resolver code"
  long_term: "Use evaluate-code API in CI/CD for pre-deployment validation"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Fix JavaScript syntax errors | YELLOW | Resolver code change |
| Ensure request/response functions exported | YELLOW | Resolver code change |
| Return valid request objects | YELLOW | Resolver code change |
| Use util.error() for error handling | YELLOW | Resolver code change |
| Use evaluate-code API for testing | GREEN | Pre-deployment validation |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **Resolver code**: JavaScript resolver logic — INTERNAL
- **Evaluate results**: Test execution output — INTERNAL
- **CloudWatch Logs**: Resolver execution details — INTERNAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER deploy untested JavaScript resolvers to production
- NEVER use unsupported JavaScript features (async/await, require)

## Phase 3 — Rollback

1. Revert resolver to previous JavaScript code version
2. Use evaluate-code API to validate the rollback code
3. Update resolver: `aws appsync update-resolver --api-id <api-id> --type-name <type> --field-name <field> --code <previous-code>`
4. Verify resolver returns expected results
5. Check CloudWatch Logs for errors after rollback

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
