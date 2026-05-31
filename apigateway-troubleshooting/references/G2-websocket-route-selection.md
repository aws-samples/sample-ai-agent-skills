---
title: "G2 — WebSocket Route Selection Issues"
description: "Diagnose WebSocket API route selection expression and message routing problems"
status: active
severity: MEDIUM
triggers:
  - "route selection"
  - "route key"
  - "$default"
  - "wrong route"
  - "message routing"
  - "action"
owner: devops-agent
objective: "Resolve WebSocket route selection issues by analyzing route selection expression and route configuration"
context: "WebSocket API uses a route selection expression (default: $request.body.action) to determine which route handles each message. The expression extracts a value from the incoming message and matches it against route keys. If no match is found, the $default route handles the message. If no $default route exists, the message is dropped silently. Route keys are case-sensitive."
---

## Phase 1 — Triage

MUST:
- Check route selection expression: `aws apigatewayv2 get-api --api-id <api-id> --query 'RouteSelectionExpression'`
- List all routes: `aws apigatewayv2 get-routes --api-id <api-id> --query 'Items[*].{RouteKey:RouteKey,RouteId:RouteId}'`
- Check if $default route exists: `aws apigatewayv2 get-routes --api-id <api-id> --query "Items[?RouteKey=='$default']"`
- Verify message format matches the route selection expression
- Check execution logs for route matching: `aws logs filter-log-events --log-group-name <log-group> --start-time <epoch-ms>`

SHOULD:
- Test with a known message format to verify routing
- Check each route's integration configuration
- Verify route keys match expected values (case-sensitive)

MAY:
- Check if route response is configured for two-way communication
- Review message payload structure

## Phase 2 — Remediate

MUST:
- Ensure route selection expression matches message structure (e.g., $request.body.action for {"action": "sendMessage"})
- Create $default route as a fallback for unmatched messages
- Ensure route keys exactly match the values extracted by the route selection expression

SHOULD:
- Use consistent message format across all clients
- Implement $default route to log unmatched messages for debugging
- Document expected message format for API consumers

MAY:
- Use custom route selection expressions for complex routing logic
- Implement message validation in route integrations

## Common Issues

- symptoms: "Messages go to $default instead of specific route"
  diagnosis: "Route selection expression does not match message structure or route key is wrong."
  resolution: "Verify expression (e.g., $request.body.action) matches the message field. Check route key case."

- symptoms: "Messages are silently dropped"
  diagnosis: "No matching route and no $default route configured."
  resolution: "Create a $default route to handle unmatched messages."

- symptoms: "Route works for some messages but not others"
  diagnosis: "Route keys are case-sensitive. Message field value case doesn't match route key."
  resolution: "Ensure route keys match exactly. 'SendMessage' != 'sendMessage'."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
```

## Escalation Conditions

- API serves production traffic from external or internal consumers
- Fix requires redeploying a production stage
- Resolution involves modifying authorizer configuration
- Issue affects multiple APIs or is account-level throttling
- Custom domain name or certificate changes are required

## Data Sensitivity

MEDIUM - API keys and usage plan configurations control access to APIs. Authorizer configurations (Lambda authorizer code, Cognito pool settings) are security-sensitive. Access logs may contain request parameters, headers, and client IPs. Mapping templates may expose internal backend URLs and data transformation logic.

## Prohibited Actions

- NEVER suggest deleting a production stage - this immediately removes the API endpoint
- NEVER suggest removing all authorizers from an API that requires authentication
- NEVER recommend disabling API key requirements on usage-plan-protected endpoints
- NEVER suggest modifying throttling limits to unlimited on a production API
- NEVER recommend deleting and recreating a custom domain name - this causes DNS propagation downtime

## Phase 3 - Rollback

1. If integration settings were changed, restore original integration configuration and redeploy
2. If WebSocket route configuration was changed, restore original route integration and redeploy
3. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "websocket_routing — <specific_cause>"
evidence:
  - type: route_selection
    content: "<route selection expression>"
  - type: routes
    content: "<configured routes and keys>"
  - type: message_format
    content: "<sample message and extracted route key>"
severity: MEDIUM
mitigation:
  immediate: "Fix route selection expression or route keys"
  long_term: "Implement $default route and message format documentation"
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
  - "NEVER suggest removing authorizers to fix access"
  - "NEVER suggest disabling throttling"
  - "NEVER suggest open CORS for production APIs"
