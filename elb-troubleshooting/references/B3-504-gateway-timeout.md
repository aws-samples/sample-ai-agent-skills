---
title: "B3 — 504 Gateway Timeout"
description: "Diagnose and resolve ELB 504 Gateway Timeout errors"
status: active
severity: HIGH
triggers:
  - "504 gateway timeout"
  - "HTTP 504"
  - "ELB timeout"
  - "request timeout"
owner: devops-agent
objective: "Identify why requests are timing out through the load balancer and restore normal response times"
context: "A 504 Gateway Timeout from an ALB indicates the target did not respond within the idle timeout period (default 60 seconds). The ALB closes the connection and returns 504 to the client. Common causes: slow backend processing, database query delays, downstream service timeouts, or misconfigured idle timeout. For NLB, idle timeout is 350 seconds for TCP and cannot be changed."
---

## Phase 1 — Triage

MUST:
- Check ALB idle timeout: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn> --query 'Attributes[?Key==\`idle_timeout.timeout_seconds\`]'`
- Review CloudWatch TargetResponseTime metric: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name TargetResponseTime --dimensions Name=TargetGroup,Value=<tg-arn-suffix> --start-time <time> --end-time <time> --period 60 --statistics p99`
- Check HTTPCode_ELB_504_Count: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_504_Count --dimensions Name=LoadBalancer,Value=<lb-arn-suffix> --start-time <time> --end-time <time> --period 60 --statistics Sum`
- Verify target application is responding: `curl -v --max-time 10 http://<target-ip>:<port><path>`
- Check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`

SHOULD:
- Check ALB access logs for the specific requests timing out (look at target_processing_time)
- Review target application logs for slow operations
- Check downstream dependencies (database, external APIs) for latency
- Verify the target's connection and request timeouts

MAY:
- Profile the slow endpoints on the target application
- Check target CPU and memory utilization during timeout periods
- Review if specific request patterns (large payloads, specific paths) correlate with timeouts

## Phase 2 — Remediate

MUST:
- If target processing is legitimately slow: increase ALB idle timeout: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <lb-arn> --attributes Key=idle_timeout.timeout_seconds,Value=<seconds>`
- If target application is hanging: investigate and fix the application bottleneck
- If downstream dependency is slow: fix the dependency or implement circuit breakers
- If target is overloaded: scale up targets or optimize the application

SHOULD:
- Set idle timeout to match the maximum expected response time plus buffer
- Implement request timeouts in the application shorter than the ALB idle timeout
- Add CloudWatch alarms for TargetResponseTime p99 exceeding thresholds

MAY:
- Implement asynchronous processing for long-running requests
- Add caching to reduce backend processing time
- Configure request-level timeouts in the application framework

## Common Issues

- symptoms: "504s for all requests after a deployment"
  diagnosis: "New application version has a bug causing slow responses or deadlocks."
  resolution: "Roll back the deployment. Investigate the application changes."

- symptoms: "504s only for specific API endpoints"
  diagnosis: "Specific endpoints have slow database queries or external API calls."
  resolution: "Optimize the slow endpoints. Add database indexes or implement caching."

- symptoms: "504s during peak traffic"
  diagnosis: "Target instances are overloaded and cannot process requests within the timeout."
  resolution: "Scale up the target group. Implement auto-scaling based on request count or latency."

- symptoms: "504s with NLB — connections dropping after 350 seconds"
  diagnosis: "NLB has a fixed 350-second idle timeout for TCP connections."
  resolution: "Implement TCP keep-alive on both client and target with intervals less than 350 seconds."

## Output Format

```yaml
root_cause: "504 Gateway Timeout — <specific_cause>"
evidence:
  - type: response_time_metrics
    content: "<TargetResponseTime p99 values>"
  - type: idle_timeout
    content: "<current ALB idle timeout setting>"
severity: HIGH
mitigation:
  immediate: "Adjust idle timeout or fix slow backend processing"
  long_term: "Implement performance monitoring, auto-scaling, and circuit breakers"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/metrics): GREEN — read-only"
  - "Phase 2 idle timeout changes: YELLOW — affects all connections through the ALB"
  - "Phase 2 application scaling: YELLOW — creates new billable resources"

## Escalation Conditions

escalation_conditions:
  - "504 errors affect more than 10% of requests"
  - "Timeout is caused by a downstream AWS service issue"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Application requires idle timeout beyond 4000 seconds (ALB maximum)"
  - "Issue requires architectural changes to resolve"

## Data Sensitivity

data_sensitivity:
  - command: "ALB access logs"
    sensitivity: MEDIUM
    contains: "Client IPs, request paths, and timing information"
  - command: "CloudWatch metrics"
    sensitivity: LOW
    contains: "Aggregated performance metrics"
  - command: "application logs"
    sensitivity: MEDIUM
    contains: "Request details and processing information"

## Prohibited Actions

prohibited_actions:
  - "NEVER set ALB idle timeout to the maximum (4000s) without understanding the impact on connection resources"
  - "NEVER ignore 504 errors as acceptable behavior without investigation"
  - "NEVER disable access logging during timeout troubleshooting"
  - "NEVER reduce idle timeout below the application's normal response time"

## Phase 3 — Rollback

- Before changing idle timeout, document the current value
- If idle timeout increase causes resource exhaustion, revert to the previous value
- If application changes worsen performance, roll back the deployment
- If scaling changes are too aggressive, adjust Auto Scaling policies
- Monitor 504 error rate and TargetResponseTime after any changes
