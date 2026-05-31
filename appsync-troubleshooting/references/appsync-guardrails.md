# AWS AppSync Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any AppSync issue.

## Guardrail 1: VTL and JavaScript Resolvers Are Completely Different Runtimes
VTL mapping templates use `$ctx`, `$util`, and Apache Velocity syntax. JavaScript resolvers (APPSYNC_JS runtime) use `ctx`, `util`, and standard JavaScript. They are NOT interchangeable. Never apply VTL syntax to a JavaScript resolver or vice versa. Check the resolver runtime before debugging.

## Guardrail 2: Resolver Timeout Is Hard-Capped at 30 Seconds
The maximum resolver timeout is 30 seconds and cannot be increased. If a resolver consistently times out, the solution is to optimize the data source query, use caching, or switch to an async pattern — not to increase the timeout.

## Guardrail 3: Subscription Endpoint Is Different from GraphQL Endpoint
Subscriptions use the real-time WebSocket endpoint (`wss://`), which is separate from the GraphQL HTTPS endpoint. Clients must connect to the correct endpoint. The real-time endpoint URL is available in the API details. Using the HTTP endpoint for subscriptions will fail silently.

## Guardrail 4: Caching Requires a Provisioned Instance and Costs Money
AppSync caching is NOT free or automatic. It requires provisioning a dedicated cache instance (t2.small through r5.12xlarge). Per-resolver caching overrides API-level caching settings. Always mention cost implications when recommending caching.

## Guardrail 5: Multiple Authorization Modes Require Directive Annotations
When an API has multiple auth modes, each type and field must specify which auth mode applies using `@aws_api_key`, `@aws_cognito_user_pools`, `@aws_iam`, `@aws_oidc`, or `@aws_auth` directives. Without directives, only the default auth mode applies. Missing directives cause unexpected "Unauthorized" errors.

## Guardrail 6: Pipeline Resolver Function Order Matters
Pipeline resolvers execute functions in the defined order. Each function's response is available to the next function via `ctx.prev.result` (JS) or `$ctx.prev.result` (VTL). Data passes between functions through `ctx.stash`. Reordering functions can break the pipeline if there are data dependencies.

## Guardrail 7: API Keys Expire and Have a Maximum Lifetime of 365 Days
API keys expire after their configured TTL (default 7 days, max 365 days). Expired keys return "Unauthorized" errors. API keys are meant for development and public access patterns — never recommend them for production user authentication.

## Guardrail 8: Merged APIs Require Schema Compatibility
Merged APIs combine schemas from multiple source APIs. If source APIs have conflicting type definitions (same type name, different fields), the merge fails. Schema changes in source APIs require re-merging. The merged API does not auto-update when source APIs change.

## Guardrail 9: DynamoDB Resolver Mapping Must Match Table Schema
VTL and JavaScript resolvers for DynamoDB must use the correct key structure (partition key and optional sort key) matching the table schema. Using the wrong key attribute names or types causes silent failures where the resolver returns null instead of an error.

## Guardrail 10: Subscription Filtering Requires Enhanced Subscription Support
Subscription filtering (server-side filtering of subscription events) requires enhanced subscriptions to be enabled on the API. Without it, all subscribers to a subscription field receive all events. Filtering expressions are defined in the resolver response mapping.

## Guardrail 11: Conflict Detection Only Applies to DataStore/Amplify Patterns
Conflict detection and resolution (Versioned, Automerge, Lambda) only applies when using AppSync with Amplify DataStore for offline sync. Standard GraphQL mutations without DataStore do not use conflict detection. Do not recommend conflict resolution for non-DataStore use cases.

## Guardrail 12: HTTP Data Sources Have No Built-in Retry or Circuit Breaker
HTTP data sources call external endpoints directly. There is no built-in retry logic, circuit breaker, or timeout configuration beyond the 30-second resolver timeout. If the external endpoint is slow or unreliable, the resolver will fail. Recommend implementing retry logic in pipeline resolver functions.
