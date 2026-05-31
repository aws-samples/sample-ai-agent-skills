---
title: "B1 — 502 Bad Gateway"
description: "Diagnose and resolve ELB 502 Bad Gateway errors"
status: active
severity: CRITICAL
triggers:
  - "502 bad gateway"
  - "HTTP 502"
  - "ELB 502"
  - "bad gateway error"
owner: devops-agent
objective: "Identify the root cause of 502 errors and restore normal traffic flow through the load balancer"
context: "A 502 Bad Gateway error from an ALB indicates the load balancer received an invalid response from the target, or the target closed the connection before sending a response. Common causes include target application crashes, connection timeouts, malformed responses, SSL/TLS handshake failures to targets, or Lambda function errors. For NLB, 502s are not generated — the connection simply fails."
---

## Phase 1 — Triage

MUST:
- Check ALB access logs for 502 errors: review access logs in S3 for `elb_status_code=502`
- Check target health: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Review CloudWatch HTTPCode_ELB_502_Count metric: `aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_502_Count --dimensions Name=LoadBalancer,Value=<lb-arn-suffix> --start-time <time> --end-time <time> --period 60 --statistics Sum`
- Check if targets are responding: `curl -v http://<target-ip>:<port><path>` (from within the VPC)
- Verify target application is running and listening on the expected port

SHOULD:
- Check target application logs for errors or crashes
- Verify the target's response headers are valid HTTP
- Check if the target is closing connections prematurely (keep-alive mismatch)
- Review ALB idle timeout vs target keep-alive timeout: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn> --query 'Attributes[?Key==\`idle_timeout.timeout_seconds\`]'`

MAY:
- Check if the 502s correlate with deployment events
- Review target CPU/memory utilization during 502 occurrences
- Check if the target group uses HTTPS and the target's TLS certificate is valid

## Phase 2 — Remediate

MUST:
- If target application crashed: restart the application and investigate the crash cause
- If keep-alive mismatch: ensure target keep-alive timeout is greater than ALB idle timeout (default 60s)
- If target returns malformed HTTP: fix the application to return valid HTTP responses
- If SSL/TLS handshake failure to target: verify the target's TLS certificate and protocol configuration

SHOULD:
- Set ALB idle timeout appropriately: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <lb-arn> --attributes Key=idle_timeout.timeout_seconds,Value=<seconds>`
- Enable ALB access logging for future debugging: `aws elbv2 modify-load-balancer-attributes --load-balancer-arn <lb-arn> --attributes Key=access_logs.s3.enabled,Value=true Key=access_logs.s3.bucket,Value=<bucket> Key=access_logs.s3.prefix,Value=<prefix>`
- Monitor HTTPCode_ELB_502_Count after remediation

MAY:
- Implement connection draining on the application side
- Add retry logic in the client for transient 502 errors
- Set up CloudWatch alarms for 502 error rate thresholds

## Common Issues

- symptoms: "Intermittent 502s during deployments"
  diagnosis: "Targets being replaced during deployment — old targets deregistering while new ones are not yet healthy."
  resolution: "Use rolling deployments with proper health check grace periods. Increase deregistration delay."

- symptoms: "502s after ALB idle timeout change"
  diagnosis: "ALB idle timeout exceeds target keep-alive timeout, causing the target to close the connection before ALB."
  resolution: "Set target keep-alive timeout higher than ALB idle timeout."

- symptoms: "502s only for specific URLs or large responses"
  diagnosis: "Target application timeout or memory issue for specific requests."
  resolution: "Investigate the specific endpoints. Check for slow database queries or memory-intensive operations."

- symptoms: "502s with HTTPS target group"
  diagnosis: "TLS handshake failure between ALB and target — expired or self-signed certificate."
  resolution: "Update the target's TLS certificate. ALB validates the certificate if using HTTPS health checks."

## Output Format

```yaml
root_cause: "502 Bad Gateway — <specific_cause>"
evidence:
  - type: access_logs
    content: "<relevant ALB access log entries>"
  - type: target_health
    content: "<target health status>"
severity: CRITICAL
mitigation:
  immediate: "Fix target application or timeout configuration"
  long_term: "Implement proper keep-alive settings, monitoring, and deployment strategies"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/logs): GREEN — read-only"
  - "Phase 2 idle timeout changes: YELLOW — affects all connections through the ALB"
  - "Phase 2 application restart: YELLOW — briefly interrupts service on that target"

## Escalation Conditions

escalation_conditions:
  - "502 error rate exceeds 50% of total requests"
  - "All targets are returning 502 errors simultaneously"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "502s are caused by upstream dependency failures"
  - "Issue requires ALB replacement or reconfiguration"

## Data Sensitivity

data_sensitivity:
  - command: "ALB access logs"
    sensitivity: MEDIUM
    contains: "Client IPs, request paths, and response codes"
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and health status"
  - command: "application logs"
    sensitivity: MEDIUM
    contains: "Application errors and request details"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable health checks to mask 502 errors"
  - "NEVER set idle timeout to 0 or extremely high values without understanding impact"
  - "NEVER remove all targets to stop 502 errors — this causes 503 instead"
  - "NEVER ignore persistent 502 errors as transient"

## Phase 3 — Rollback

- Before changing ALB attributes, document current settings: `aws elbv2 describe-load-balancer-attributes --load-balancer-arn <lb-arn>`
- If idle timeout changes worsen the issue, revert to the previous value
- If application changes cause more errors, roll back the application deployment
- If target group configuration was modified, restore previous settings
- Monitor error rates for 15 minutes after any change to confirm improvement
