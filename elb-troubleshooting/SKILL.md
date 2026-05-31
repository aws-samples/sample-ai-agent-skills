---
name: elb-diagnostics
version: "1.0.0"
last_updated: "2025-04-12"
description: >
  Use this skill to investigate and troubleshoot Elastic Load Balancing
  problems (ALB, NLB, CLB) by analyzing load balancer configurations,
  target health, listener rules, and following structured runbooks.
  Activate when: 5xx errors, health check failures, target registration
  issues, SSL/TLS problems, routing errors, slow responses, connection
  draining issues, WAF blocks, or the user says something is wrong
  with their load balancer without naming specific symptoms.
compatibility: >
  Requires AWS CLI or SDK access with ELBv2, EC2, ACM, WAF,
  CloudWatch, and optionally S3 (access logs) permissions.
---

# ELB Diagnostics

## When to use

Any ELB investigation where the console alone is insufficient — 5xx error analysis, health check debugging, SSL/TLS troubleshooting, routing rule evaluation, NLB source IP issues, or performance optimization.

## Investigation workflow

### Step 1 — Collect and triage

```
aws elbv2 describe-load-balancers --load-balancer-arns <lb-arn>
aws elbv2 describe-listeners --load-balancer-arn <lb-arn>
aws elbv2 describe-target-groups --load-balancer-arn <lb-arn>
aws elbv2 describe-target-health --target-group-arn <tg-arn>
aws elbv2 describe-rules --listener-arn <listener-arn>
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn>
```

### Step 2 — Domain deep dive

```
aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn>
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_5XX_Count ...
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name TargetResponseTime ...
aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount ...
aws acm describe-certificate --certificate-arn <cert-arn>
```

Read `references/elb-guardrails.md` before concluding on any ELB issue.

## Tool quick reference

| Tool / API | When to use |
|------------|-------------|
| `describe-load-balancers` | LB type, scheme, AZs, security groups |
| `describe-listeners` | Listener protocol, port, certificates |
| `describe-target-groups` | Target type, health check config |
| `describe-target-health` | Individual target health status |
| `describe-rules` | Routing rules, conditions, actions |
| `describe-load-balancer-attributes` | Idle timeout, access logs, cross-zone |
| `describe-target-group-attributes` | Deregistration delay, stickiness |

## Gotchas: ELB

- ALB operates at Layer 7 (HTTP/HTTPS). NLB operates at Layer 4 (TCP/UDP/TLS). They have fundamentally different behaviors and troubleshooting approaches.
- Health check grace period matters for new targets. Without it, targets may be marked unhealthy before the application finishes starting.
- Deregistration delay (default 300 seconds) keeps existing connections alive during deregistration. In-flight requests complete but no new requests are sent.
- Cross-zone load balancing is enabled by default on ALB (no extra cost). On NLB, it's disabled by default and incurs cross-AZ data transfer costs when enabled.
- ALB idle timeout (default 60 seconds) must be LESS than the backend application's keep-alive timeout. If the backend closes first, ALB returns 502.
- NLB preserves the client source IP by default. ALB does NOT — use X-Forwarded-For header. Proxy Protocol v2 is available on NLB for TCP listeners.
- Security groups exist on ALB but NOT on NLB by default (NLB can optionally have SGs). NLB targets must allow traffic from client IPs directly.
- Sticky sessions use cookies on ALB (application or duration-based) and source IP on NLB. ALB stickiness is per target group.
- Target group health check path must return HTTP 200 (or configured success codes). A health check path that returns 301/302 will fail.
- ALB supports WebSocket connections natively. The connection upgrade happens automatically. Idle timeout applies to WebSocket connections.

### Load balancer comparison

| Feature | ALB | NLB | CLB |
|---------|-----|-----|-----|
| Layer | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 4/7 |
| Source IP | X-Forwarded-For | Preserved | X-Forwarded-For |
| Security Groups | Yes | Optional | Yes |
| Cross-Zone | Default on (free) | Default off (costs) | Configurable |
| Sticky Sessions | Cookie-based | Source IP | Cookie-based |
| WebSocket | Native | TCP passthrough | No |

## Anti-hallucination rules

1. Always cite specific load balancer configurations, target health, or CloudWatch metrics as evidence.
2. ALB and NLB have different behaviors. Never apply ALB troubleshooting to NLB or vice versa without noting the differences.
3. NLB preserves source IP by default. Never claim NLB uses X-Forwarded-For.
4. Cross-zone on NLB costs money. Never recommend enabling it without mentioning the cost impact.
5. 502 errors from ALB usually mean the backend closed the connection. Never blame the load balancer without checking backend keep-alive settings.
6. Spend no more than 2 minutes on any single hypothesis. Pivot if inconclusive.

## 30 runbooks

| Category | IDs | Covers |
|----------|-----|--------|
| A — Health Checks | A1-A4 | Failing health checks, grace period, unhealthy targets, config |
| B — Errors | B1-B4 | 502, 503, 504, 4xx errors |
| C — SSL/TLS | C1-C3 | Certificate errors, TLS negotiation, backend SSL |
| D — Routing | D1-D3 | Path-based, host-based, fixed response |
| E — NLB-Specific | E1-E3 | Source IP preservation, cross-zone, TCP health checks |
| F — Performance | F1-F3 | Slow targets, connection draining, idle timeout |
| G — Target Groups | G1-G2 | Registration, deregistration |
| H — Integration | H1-H2 | WAF, access logs |
| Z — Catch-All | Z1 | General troubleshooting |
