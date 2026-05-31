# Amazon API Gateway Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any API Gateway issue.

## Guardrail 1: 29-Second Integration Timeout Is a Hard Limit (REST API)
REST API has a maximum integration timeout of 29 seconds. This CANNOT be increased — it is a hard service limit. HTTP API has a 30-second maximum. If the backend needs more time, use asynchronous patterns: invoke Lambda asynchronously, use Step Functions, or queue via SQS. Never tell users to increase the timeout beyond 29 seconds.

## Guardrail 2: Deployment Required After REST API Changes
REST API changes to methods, integrations, authorizers, models, and gateway responses are NOT live until a new deployment is created. Forgetting to deploy is the #1 cause of "my changes aren't working." HTTP API has automatic deployments enabled by default via the $default stage. Always ask: "Did you create a new deployment after making changes?"

## Guardrail 3: CORS Requires OPTIONS Method and Response Headers
For REST API, CORS requires: (1) an OPTIONS method with MOCK integration returning Access-Control-Allow-Origin, Access-Control-Allow-Methods, Access-Control-Allow-Headers headers, AND (2) the actual method responses must also include Access-Control-Allow-Origin. Gateway responses (4xx/5xx) also need CORS headers — configure them in gateway responses. HTTP API has a built-in CORS configuration object.

## Guardrail 4: API Keys Are NOT for Authentication
API keys provide identification and usage tracking, NOT authentication or authorization. They are sent in the x-api-key header and associated with usage plans for throttling and quotas. For authentication, use Lambda authorizers, Cognito user pool authorizers, or IAM authorization. Never recommend API keys as a security mechanism.

## Guardrail 5: Authorizer Caching Can Cause Stale Auth Decisions
Lambda authorizer results are cached based on the token or identity sources (TTL 0-3600 seconds, default 300s). A cached ALLOW continues to allow access even if the user's permissions changed. A cached DENY blocks access even after fixing the authorizer. Set TTL to 0 during debugging. The cache key is the authorization token value — different tokens get separate cache entries.

## Guardrail 6: Throttling Hierarchy — Account > Stage > Method > Usage Plan
Account-level default: 10,000 rps with 5,000 burst capacity. Stage-level and method-level overrides can only REDUCE limits, never exceed account limits. Usage plans add per-client throttling via API keys. When diagnosing 429 errors, check ALL levels. A method override of 100 rps means that method is limited to 100 rps regardless of account limits.

## Guardrail 7: Lambda Proxy Response Format Is Strict
Lambda proxy integration requires the Lambda function to return a specific JSON format: { statusCode: number, headers: object, body: string }. The body MUST be a string (use JSON.stringify for objects). Missing statusCode or non-string body causes a 502 "Internal server error" from API Gateway. Multivalue headers use multiValueHeaders field.

## Guardrail 8: Stage Variables Are Not Environment Variables
Stage variables are API Gateway configuration values available in integration URIs, Lambda function ARNs, and mapping templates via $stageVariables. They are NOT Lambda environment variables. They are resolved at request time by API Gateway. Use them to point different stages to different Lambda aliases or backend URLs.

## Guardrail 9: Binary Media Types Require Explicit Configuration
Binary media types must be listed in the API's binaryMediaTypes setting. For Lambda proxy, when Content-Type matches a binary media type, the body is base64-encoded. The Lambda response must set isBase64Encoded: true and the body must be base64-encoded. The Accept header from the client must also match. Missing any of these causes garbled responses.

## Guardrail 10: Custom Domain ACM Certificate Region Matters
Edge-optimized custom domains require the ACM certificate in us-east-1 (because CloudFront is global). Regional custom domains require the ACM certificate in the SAME region as the API. Using a certificate from the wrong region causes domain creation to fail. Always verify the certificate region matches the endpoint type.

## Guardrail 11: WebSocket Route Selection Expression Determines Message Routing
The route selection expression (default: $request.body.action) extracts the route key from incoming messages. If the expression evaluates to null or doesn't match any route, the $default route handles the message. If no $default route exists, the message is dropped silently. The $connect and $disconnect routes are invoked automatically on connection lifecycle events.

## Guardrail 12: VPC Link Requirements Differ Between REST and HTTP API
REST API VPC links (v1) require a Network Load Balancer (NLB) in the target VPC. HTTP API VPC links (v2) support ALB, NLB, or AWS Cloud Map. VPC links must be in AVAILABLE status — PENDING or FAILED links cause integration errors. VPC link creation can take several minutes. Always verify VPC link status before testing integrations.
