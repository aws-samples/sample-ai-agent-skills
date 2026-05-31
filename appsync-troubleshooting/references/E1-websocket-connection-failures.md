---
title: "E1 — WebSocket Connection Failures"
description: "Diagnose AppSync real-time subscription WebSocket connection failures"
status: active
severity: HIGH
triggers:
  - "WebSocket"
  - "subscription connection"
  - "wss://"
  - "real-time connection"
  - "subscription not working"
  - "connection refused"
owner: devops-agent
objective: "Identify and resolve WebSocket connection failures for AppSync subscriptions"
context: "AppSync subscriptions use WebSocket connections via the real-time endpoint (wss://). The connection uses MQTT over WebSocket protocol. Common issues include using the wrong endpoint, authorization failures during connection handshake, client library misconfiguration, connection limits (100 subscriptions per connection), and network/firewall blocking WebSocket traffic."
---

## Phase 1 — Triage

MUST:
- Get the real-time endpoint: `aws appsync get-graphql-api --api-id <api-id> --query 'graphqlApi.uris'`
- Verify the client is connecting to the `wss://` real-time endpoint (NOT the HTTPS GraphQL endpoint)
- Check the authorization mode and verify credentials are passed during WebSocket handshake
- Check CloudWatch metrics for connection errors: `aws cloudwatch get-metric-statistics --namespace AWS/AppSync --metric-name ConnectServerError --dimensions Name=GraphQLAPIId,Value=<api-id> --start-time <start> --end-time <end> --period 300 --statistics Sum`
- Check CloudWatch Logs for subscription connection errors

SHOULD:
- Verify the client library supports AppSync WebSocket protocol (Amplify, graphql-ws with AppSync adapter)
- Check if the connection limit (100 subscriptions per connection) is being exceeded
- Verify network/firewall allows WebSocket (wss://) connections on port 443

MAY:
- Test WebSocket connectivity using wscat: `wscat -c "wss://<realtime-endpoint>" -H "host:<realtime-host>"`
- Check if a WAF WebACL is blocking WebSocket connections
- Verify DNS resolution for the real-time endpoint

## Phase 2 — Remediate

MUST:
- Use the correct real-time wss:// endpoint (not the GraphQL HTTPS endpoint)
- Pass valid authorization credentials during the WebSocket connection handshake
- Use a compatible client library (Amplify JS, graphql-ws with AppSync link)

SHOULD:
- Implement connection retry logic with exponential backoff
- Handle connection lifecycle events (connect, disconnect, error)
- Keep connections alive with periodic keepalive messages (5-minute idle timeout)

MAY:
- Use multiple WebSocket connections if exceeding 100 subscriptions per connection
- Implement connection pooling for high-subscription-count applications

## Common Issues

- symptoms: "WebSocket connection immediately closes"
  diagnosis: "Client is connecting to the GraphQL HTTPS endpoint instead of the wss:// real-time endpoint."
  resolution: "Use the real-time endpoint from get-graphql-api uris.REALTIME."

- symptoms: "Connection handshake fails with 401"
  diagnosis: "Authorization credentials are missing or invalid during WebSocket handshake."
  resolution: "Pass valid auth credentials (API key, JWT, SigV4) in the connection_init message."

- symptoms: "Connection drops after 5 minutes of inactivity"
  diagnosis: "WebSocket idle timeout. No keepalive messages sent."
  resolution: "Implement keepalive ping messages within the 5-minute idle timeout window."

## Output Format

```yaml
root_cause: "websocket_connection_failure — <specific_cause>"
evidence:
  - type: api_endpoints
    content: "<GraphQL and real-time endpoint URLs>"
  - type: connection_metrics
    content: "<ConnectServerError metrics>"
severity: HIGH
mitigation:
  immediate: "Fix endpoint URL, authorization, or client configuration"
  long_term: "Implement connection lifecycle management and retry logic"
```


## Safety Ratings

| Phase 2 Action | safety_rating | Rationale |
|---|---|---|
| Use correct real-time wss:// endpoint | GREEN | Client configuration fix |
| Pass valid auth credentials in handshake | GREEN | Client configuration fix |
| Use compatible client library | GREEN | Client library selection |
| Implement connection retry logic | YELLOW | Client code change |
| Handle connection lifecycle events | YELLOW | Client code change |
| Keep connections alive with keepalive | YELLOW | Client code change |

## Escalation Conditions

- Fix requires modifying production resolver
- Fix requires changing auth mode

## Data Sensitivity

- **API endpoints**: GraphQL and real-time URLs — INTERNAL
- **Connection metrics**: WebSocket connection errors — INTERNAL
- **Auth credentials**: Tokens passed during handshake — CONFIDENTIAL

## Prohibited Actions

- NEVER suggest removing authorization to fix access denied
- NEVER expose WebSocket endpoints without authentication
- NEVER disable TLS on WebSocket connections

## Phase 3 — Rollback

1. Revert client library changes to previous version
2. Revert connection configuration to previous settings
3. Verify WebSocket connections are established successfully
4. Monitor connection error metrics after rollback
5. Verify subscriptions are receiving events

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
