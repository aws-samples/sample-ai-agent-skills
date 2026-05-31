---
name: appsync-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot AWS AppSync
  problems (GraphQL APIs, merged APIs, real-time subscriptions) by
  analyzing API configurations, resolvers, data sources, authorization,
  and following structured runbooks.
  Activate when: schema errors, resolver failures, VTL mapping issues,
  JavaScript resolver errors, subscription disconnects, auth failures,
  caching problems, query latency, N+1 resolver issues, or the user
  says something is wrong with their AppSync API without naming
  specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with AppSync, DynamoDB, Lambda,
  CloudWatch, IAM, Cognito, and optionally RDS/Aurora permissions.
---

# AppSync Diagnostics

## When to use

Any AppSync investigation where the console alone is insufficient — resolver errors, schema deployment failures, authorization issues, subscription connection problems, caching misconfigurations, query latency, N+1 data fetching, or merged API conflicts.

## Investigation workflow

### Step 1 — Collect and triage

```
aws appsync list-graphql-apis
aws appsync get-graphql-api --api-id <api-id>
aws appsync list-resolvers --api-id <api-id> --type-name <type>
aws appsync list-data-sources --api-id <api-id>
aws appsync get-schema-creation-status --api-id <api-id>
aws appsync list-api-keys --api-id <api-id>
```

### Step 2 — Domain deep dive

```
aws appsync get-resolver --api-id <api-id> --type-name <type> --field-name <field>
aws appsync get-data-source --api-id <api-id> --name <ds-name>
aws cloudwatch get-metric-statistics --namespace AWS/AppSync --metric-name 4XXError --dimensions Name=GraphQLAPIId,Value=<api-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/AppSync --metric-name 5XXError --dimensions Name=GraphQLAPIId,Value=<api-id> ...
aws cloudwatch get-metric-statistics --namespace AWS/AppSync --metric-name Latency --dimensions Name=GraphQLAPIId,Value=<api-id> ...
aws appsync list-functions --api-id <api-id>
```

Read `references/appsync-guardrails.md` before concluding on any AppSync issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `list-graphql-apis` | List all APIs in the account/region |
| `get-graphql-api` | API config, auth modes, log settings, endpoints |
| `list-resolvers` | Resolvers for a given type |
| `get-resolver` | Resolver runtime, mapping templates, pipeline config |
| `list-data-sources` | All data sources attached to the API |
| `get-data-source` | Data source type, config, IAM role |
| `get-schema-creation-status` | Schema deployment status and errors |
| `list-api-keys` | API key expiration and status |
| `list-functions` | Pipeline functions for pipeline resolvers |

## Gotchas: AppSync

- VTL mapping templates (Apache Velocity) vs JavaScript resolvers (APPSYNC_JS runtime) are two distinct resolver runtimes. VTL uses `$ctx` and `$util`, JavaScript uses `ctx` and `util` — they are NOT interchangeable.
- Pipeline resolvers chain multiple functions (before → function1 → function2 → ... → after). Each function has its own request/response mapping. The `$ctx.stash` (VTL) or `ctx.stash` (JS) passes data between functions.
- Resolver timeout is 30 seconds maximum. This cannot be increased. For long-running operations, use async patterns with DynamoDB or EventBridge.
- Subscriptions connect via WebSocket using the `wss://` endpoint (realtime endpoint). The connection URL is different from the GraphQL HTTP endpoint. Clients must use `graphql-ws` or Amplify libraries.
- Real-time subscriptions use MQTT over WebSocket protocol internally. Connection limits apply: 100 subscriptions per connection, connection idle timeout of 5 minutes without keepalive.
- Five authorization modes: API_KEY, AMAZON_COGNITO_USER_POOLS, AWS_IAM, OPENID_CONNECT, AWS_LAMBDA. Multiple auth modes can be configured on a single API with `@aws_auth` directives.
- Caching TTL is configurable per API (1–3600 seconds). Per-resolver caching overrides API-level caching. Caching requires a dedicated cache instance (t2.small to r5.12xlarge) — it is NOT free.
- Conflict detection for offline/real-time sync: Versioned (optimistic concurrency), Automerge (automatic conflict resolution), Lambda (custom resolution logic). Only applies when using AppSync with DataStore/Amplify.
- Merged APIs combine multiple source APIs into a single endpoint. Schema conflicts between source APIs cause merge failures. Source API changes require re-merge.
- Batch resolvers (BatchInvoke for Lambda, BatchGetItem for DynamoDB) solve the N+1 problem. Without batching, a list query with N items triggers N individual resolver invocations.

### Authorization mode comparison

| Mode | Use Case | Token Location | Expiry |
|------|----------|---------------|--------|
| API_KEY | Public/dev access | `x-api-key` header | Max 365 days |
| COGNITO | User-based auth | `Authorization` header (JWT) | Token-based |
| IAM | Service-to-service | SigV4 signed request | Session-based |
| OIDC | External IdP | `Authorization` header (JWT) | Token-based |
| LAMBDA | Custom auth logic | Custom token header | Per-request |

## Anti-hallucination rules

1. Always cite specific API configurations, resolver code, CloudWatch metrics, or error messages as evidence.
2. VTL and JavaScript resolvers have completely different syntax and utilities. Never mix VTL `$util` with JavaScript `util` or vice versa.
3. Resolver timeout is hard-capped at 30 seconds. Never suggest increasing it beyond 30s.
4. Subscription endpoints use `wss://` and are different from the GraphQL HTTPS endpoint. Never use the HTTP endpoint for subscriptions.
5. Caching requires a provisioned cache instance with associated costs. Never claim caching is free or automatic.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 26 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — API | A1–A3 | API creation failures, schema errors, deployment issues |
| B — Resolvers | B1–B4 | VTL errors, JavaScript resolver errors, pipeline failures, timeout |
| C — Data Sources | C1–C4 | DynamoDB, Lambda, HTTP, RDS/Aurora data source errors |
| D — Auth | D1–D3 | API key issues, Cognito auth, IAM/OIDC auth |
| E — Subscriptions | E1–E2 | WebSocket connection failures, subscription filtering |
| F — Caching | F1–F2 | Cache configuration, cache invalidation |
| G — Performance | G1–G2 | Query latency, N+1 resolver issues |
| Z — Catch-All | Z1 | General troubleshooting |
