# Edge Functions Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CloudFront Functions or Lambda@Edge issue.

## Guardrail 1: Two Different Compute Models
CloudFront Functions run at 200+ edge locations (viewer-facing). Lambda@Edge runs at regional edge caches (13 regions). They have fundamentally different capabilities, limits, and pricing. Do not treat them as interchangeable.

## Guardrail 2: CloudFront Functions Are Viewer-Only
CloudFront Functions can only be associated with viewer-request and viewer-response events. They CANNOT be used for origin-request or origin-response events. Only Lambda@Edge supports all four event types.

## Guardrail 3: Lambda@Edge Must Be in us-east-1
Lambda@Edge functions MUST be created in the us-east-1 (N. Virginia) region. CloudFront replicates them to edge locations automatically. Functions created in other regions cannot be associated with CloudFront distributions.

## Guardrail 4: CloudFront Functions Have a 1ms Time Limit
CloudFront Functions have a maximum execution time of 1 millisecond. This is a hard limit — functions exceeding it are terminated and the request fails. Only lightweight operations (URL rewrites, header manipulation, simple redirects) are feasible.

## Guardrail 5: Lambda@Edge Has Different Timeouts Per Event Type
Lambda@Edge viewer events (viewer-request, viewer-response) have a 5-second timeout. Origin events (origin-request, origin-response) have a 30-second timeout. These are hard limits that cannot be increased.

## Guardrail 6: Lambda@Edge Restrictions
Lambda@Edge functions cannot use: environment variables, Lambda layers, VPC configuration, dead letter queues, provisioned concurrency, EFS file systems, or container images. These are permanent restrictions, not configuration issues.

## Guardrail 7: CloudFront Functions Cannot Make Network Calls
CloudFront Functions run in a restricted JavaScript runtime that does NOT support network calls (no fetch, no HTTP requests, no DNS lookups). All data must be available in the request/response event. Lambda@Edge can make network calls in origin events.

## Guardrail 8: Function Associations Are Per Cache Behavior
Functions are associated with specific cache behaviors, not with the entire distribution. Different URL path patterns can use different functions. A single cache behavior can have one CloudFront Function and one Lambda@Edge function per event type.

## Guardrail 9: CloudFront Functions Use ECMAScript 5.1
CloudFront Functions use a JavaScript runtime compatible with ECMAScript 5.1. Modern JavaScript features (async/await, arrow functions in some contexts, destructuring) may not work. Test thoroughly before publishing.

## Guardrail 10: Lambda@Edge Replication Takes Time
After deploying or updating a Lambda@Edge function, replication to all edge locations takes several minutes (typically 5-15 minutes, sometimes longer). During replication, different edge locations may run different versions.

## Guardrail 11: CloudFront Function Publish vs Development Stage
CloudFront Functions have two stages: DEVELOPMENT and LIVE. Functions must be explicitly published from DEVELOPMENT to LIVE. Testing in DEVELOPMENT does not affect production traffic. Only LIVE functions serve real requests.

## Guardrail 12: Response Body Manipulation Limits
CloudFront Functions cannot access or modify response bodies. Lambda@Edge can generate response bodies (up to 1MB for viewer events, 40KB for origin events when including body). Neither can modify origin response bodies in-flight.
