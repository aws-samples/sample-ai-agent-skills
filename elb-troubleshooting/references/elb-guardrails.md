# ELB Anti-Misdiagnosis Guardrails

Read these guardrails BEFORE concluding on any ELB issue.

## Guardrail 1: ALB and NLB Are Fundamentally Different
ALB operates at Layer 7 (HTTP/HTTPS). NLB operates at Layer 4 (TCP/UDP/TLS). Troubleshooting approaches differ significantly. Do not apply ALB-specific advice to NLB or vice versa.

## Guardrail 2: 502 Usually Means Backend Closed Connection
ALB 502 errors most commonly occur when the backend closes the TCP connection before the ALB. Ensure the backend keep-alive timeout is GREATER than the ALB idle timeout (default 60s).

## Guardrail 3: Health Check Grace Period Prevents False Unhealthy
New targets need time to start. Without a health check grace period, targets may be marked unhealthy and removed before the application finishes initializing. Always check if a grace period is configured.

## Guardrail 4: NLB Preserves Source IP by Default
NLB preserves the client's source IP. Target security groups must allow traffic from client IPs, not from the NLB. ALB does NOT preserve source IP — use X-Forwarded-For header.

## Guardrail 5: Cross-Zone on NLB Has Cost Implications
Cross-zone load balancing on NLB incurs cross-AZ data transfer charges. On ALB, it's enabled by default at no extra cost. Never recommend enabling NLB cross-zone without mentioning costs.

## Guardrail 6: Deregistration Delay Keeps Connections Alive
Default deregistration delay is 300 seconds. During this time, existing connections complete but no new requests are sent. Reducing this too aggressively can drop in-flight requests.

## Guardrail 7: Health Check Path Must Return 200
The health check path must return HTTP 200 (or the configured success codes). A path that returns 301/302 redirects will cause health checks to fail. Check the actual response code of the health check path.

## Guardrail 8: ALB Idle Timeout Must Be Less Than Backend Keep-Alive
If the backend's keep-alive timeout is less than the ALB idle timeout, the backend may close connections that the ALB considers active, causing 502 errors.

## Guardrail 9: Security Groups on ALB, Not Default on NLB
ALB requires security groups. NLB does not have security groups by default (optional since 2023). NLB targets must allow traffic from client IPs directly.

## Guardrail 10: Sticky Sessions Differ Between ALB and NLB
ALB uses cookie-based stickiness (application or duration-based). NLB uses source IP-based stickiness. They are configured differently and have different behaviors.

## Guardrail 11: Target Group Health Check Is Independent
Target group health checks are independent of the application's actual health endpoint. A target can pass health checks but still return errors to real traffic if the health check path is too simple.

## Guardrail 12: WAF Rules Can Cause Unexpected 403s
When WAF is associated with an ALB, WAF rules are evaluated before routing. A WAF rule blocking legitimate traffic appears as a 403 from the ALB, not from the backend.
