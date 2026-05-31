# Amazon CloudFront Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any CloudFront issue.

## Guardrail 1: TTL Hierarchy — Origin Headers Can Override CloudFront Settings
CloudFront TTL resolution follows: max(MinTTL, min(MaxTTL, origin-header-or-DefaultTTL)). If the origin sends Cache-Control: max-age=0, CloudFront still caches for MinTTL seconds. If MinTTL is 0 and the origin sends no cache headers, DefaultTTL applies. Never assume CloudFront settings alone control caching — always check origin response headers.

## Guardrail 2: Invalidations Are NOT Instant
Invalidations propagate to all edge locations worldwide. They typically complete in 1-2 minutes but can take up to 10-15 minutes. Do not tell users content is immediately updated after creating an invalidation. The first 1,000 paths per month are free; after that, $0.005 per path. Recommend versioned file names (e.g., style.v2.css) over invalidations for frequent updates.

## Guardrail 3: OAC Is the Recommended Method — OAI Is Legacy
Origin Access Control (OAC) replaces Origin Access Identity (OAI). OAC supports SSE-KMS encrypted objects, all S3 regions including opt-in regions, S3 Object Lambda, and dynamic origins. OAI does not support SSE-KMS, newer S3 features, or opt-in regions. Always recommend OAC for new distributions.

## Guardrail 4: Cache Behavior Path Matching — First Match Wins, Not Longest
Cache behaviors are evaluated in the order they appear in the distribution configuration. The first path pattern that matches the request wins. The default behavior (*) is always evaluated last. This is NOT longest-match like ALB path routing. If /images/* appears before /images/thumbnails/*, the more specific pattern will never match.

## Guardrail 5: Origin Failover Only Triggers on 5xx and Connection Failures
Origin groups with failover only switch to the secondary origin when the primary returns HTTP 500, 502, 503, or 504, or when CloudFront cannot connect to the primary origin. 4xx errors (403, 404) do NOT trigger failover. Do not recommend origin failover as a solution for 4xx errors.

## Guardrail 6: Lambda@Edge Limits — 5s Viewer, 30s Origin, us-east-1 Only
Lambda@Edge viewer request/response functions: 5-second timeout, 128 MB memory, 40 KB response body (viewer response). Origin request/response functions: 30-second timeout, up to 10 GB memory, 1 MB response body (origin response). All functions MUST be deployed in us-east-1. No environment variables, no VPC access, no layers, no ARM architecture, no EFS.

## Guardrail 7: CloudFront Functions Limits — 10ms Execution, No Network Access
CloudFront Functions have a 10 ms maximum execution time, 2 MB maximum function size, and 10 KB maximum response body for generated responses. They have NO network access, NO file system access, and only support viewer request and viewer response events. They cannot modify origin requests/responses. Use Lambda@Edge for anything requiring network calls or origin event triggers.

## Guardrail 8: Signed URLs vs Signed Cookies — Different Use Cases
Signed URLs restrict access to individual files and change the URL. Signed cookies restrict access to multiple files without changing URLs (ideal for HLS/DASH streaming). Both require a trusted key group (recommended) or CloudFront key pair (legacy). Key pairs are managed in the root account only — use trusted key groups instead.

## Guardrail 9: Custom Error Pages Are Cached Separately
Custom error responses have their own TTL (Error Caching Minimum TTL, default 10 seconds). Changing the custom error page or the origin error does NOT immediately update what viewers see — the cached error response must expire or be invalidated. Always check the error caching TTL when debugging stale error pages.

## Guardrail 10: Default Root Object Only Applies to the Root URL
The default root object (e.g., index.html) ONLY applies to requests for the distribution root (https://d111.cloudfront.net/). It does NOT apply to subdirectory requests (https://d111.cloudfront.net/subdir/). For subdirectory index documents, use Lambda@Edge or CloudFront Functions to append index.html to directory requests.

## Guardrail 11: HTTP Methods Are Configured Per Cache Behavior
Each cache behavior independently configures allowed HTTP methods. Options are: GET/HEAD, GET/HEAD/OPTIONS, or GET/HEAD/OPTIONS/PUT/POST/PATCH/DELETE. If POST requests fail with 403, check that the matching cache behavior allows POST. The default behavior may only allow GET/HEAD.

## Guardrail 12: Distribution Deployment Takes Time
Distribution changes (new behaviors, origin changes, SSL certificates) require deployment to all edge locations. This typically takes 5-10 minutes but can take up to 30 minutes. The distribution status shows "InProgress" during deployment. Do not assume changes are live immediately after API call returns. Check distribution status before testing.
