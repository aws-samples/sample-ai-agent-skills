---
title: "G3 — WebSocket Message Delivery Issues"
description: "Diagnose WebSocket API server-to-client message delivery failures via callback URL"
status: active
severity: HIGH
triggers:
  - "message delivery"
  - "callback URL"
  - "@connections"
  - "post to connection"
  - "GoneException"
  - "send message"
owner: devops-agent
objective: "Resolve WebSocket message delivery issues including callback URL configuration, connection management, and permissions"
context: "WebSocket API server-to-client messaging uses the callback URL: https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/@connections/<connection-id>. The backend POSTs messages to this URL. Common failures include stale connection IDs (GoneException), missing execute-api:ManageConnections permission, incorrect callback URL format, and payload size limits (128 KB for WebSocket frames)."
---

## Phase 1 — Triage

MUST:
- Verify callback URL format: `https://<api-id>.execute-api.<region>.amazonaws.com/<stage>/@connections/<connection-id>`
- Check if connection is still active (GoneException means disconnected)
- Verify Lambda/backend has execute-api:ManageConnections permission
- Check Lambda logs for callback URL errors: `aws logs filter-log-events --log-group-name /aws/lambda/<function-name> --filter-pattern "GoneException" --start-time <epoch-ms>`
- Check message payload size (max 128 KB per frame)

SHOULD:
- Verify connection tracking (DynamoDB table with connection IDs)
- Check $disconnect route cleans up stale connections
- Test callback URL directly: `aws apigatewayv2 post-to-connection --api-id <api-id> --stage <stage> --connection-id <conn-id> --data '{"message":"test"}'` (AWS CLI v2)

MAY:
- Check WebSocket API CloudWatch metrics for message counts
- Review connection idle timeout (10 minutes default, max 10 minutes)

## Phase 2 — Remediate

MUST:
- Grant execute-api:ManageConnections permission to the backend:
  ```json
  {
    "Effect": "Allow",
    "Action": "execute-api:ManageConnections",
    "Resource": "arn:aws:execute-api:<region>:<account>:<api-id>/<stage>/POST/@connections/*"
  }
  ```
- Handle GoneException by removing stale connection IDs from tracking store
- Ensure callback URL uses the correct API ID, region, and stage

SHOULD:
- Implement connection tracking in DynamoDB with TTL for automatic cleanup
- Handle connection idle timeout (10 minutes) with heartbeat messages
- Implement retry logic for transient delivery failures

MAY:
- Use @connections GET to check connection status before sending
- Implement fan-out pattern for broadcasting to multiple connections

## Common Issues

- symptoms: "GoneException when posting to connection"
  diagnosis: "Client has disconnected but connection ID is still in tracking store."
  resolution: "Handle GoneException by removing connection ID. Implement $disconnect cleanup."

- symptoms: "403 Forbidden when posting to callback URL"
  diagnosis: "Backend lacks execute-api:ManageConnections permission."
  resolution: "Add execute-api:ManageConnections to the Lambda execution role."

- symptoms: "Messages not received by client despite successful POST"
  diagnosis: "Payload exceeds 128 KB frame limit or message format issue."
  resolution: "Check payload size. Ensure message is properly formatted for the client."


## Safety Ratings

```
safety_ratings:
  - "Inspect/Describe resources: GREEN - Read-only API calls, no state change"
  - "Review logs and metrics: GREEN - Read-only observability data access"
  - "Modify IAM/permissions: RED - Security-sensitive, may break access patterns"
  - "Certificate/TLS changes: RED - May cause downtime if misconfigured"
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

1. If a deployment was made, redeploy the previous working deployment to the stage: `aws apigateway create-deployment --rest-api-id <id> --stage-name <stage>`
2. If integration settings were changed, restore original integration configuration and redeploy
3. If WebSocket route configuration was changed, restore original route integration and redeploy
4. If mapping templates or validators were changed, restore original templates and redeploy
## Output Format

```yaml
root_cause: "websocket_delivery — <specific_cause>"
evidence:
  - type: callback_url
    content: "<callback URL and response>"
  - type: permissions
    content: "<execute-api:ManageConnections policy>"
  - type: connection_status
    content: "<connection tracking and status>"
severity: HIGH
mitigation:
  immediate: "Fix permissions, connection tracking, or callback URL"
  long_term: "Implement robust connection management with DynamoDB and heartbeats"
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
