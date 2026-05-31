---
title: "F2 — Connection Draining"
description: "Diagnose and resolve ELB connection draining and deregistration delay issues"
status: active
severity: MEDIUM
triggers:
  - "connection draining"
  - "deregistration delay"
  - "draining timeout"
  - "in-flight requests"
  - "graceful shutdown"
owner: devops-agent
objective: "Resolve connection draining issues and ensure graceful target deregistration"
context: "Connection draining (deregistration delay) allows in-flight requests to complete before a target is fully deregistered from the load balancer. ALB/NLB default deregistration delay is 300 seconds. During draining, the LB stops sending new requests to the target but allows existing connections to complete. Issues arise from timeout values being too short (dropping requests) or too long (delaying deployments), and from applications not handling graceful shutdown properly."
---

## Phase 1 — Triage

MUST:
- Check deregistration delay: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?Key==\`deregistration_delay.timeout_seconds\`]'`
- Check for targets in draining state: `aws elbv2 describe-target-health --target-group-arn <tg-arn>` (look for State=draining)
- Verify the connection draining behavior matches expectations
- Check if deployments are waiting on draining targets
- Review CloudWatch metrics for connection counts during deregistration

SHOULD:
- Check if the application handles SIGTERM for graceful shutdown
- Verify long-lived connections (WebSocket, gRPC streaming) are handled during draining
- Review deployment pipeline timeout settings vs deregistration delay

MAY:
- Check if connection draining is causing deployment slowness
- Review ALB access logs for requests during the draining period
- Verify NLB connection draining behavior (NLB waits for active TCP connections to close)

## Phase 2 — Remediate

MUST:
- If draining too slow for deployments: reduce deregistration delay: `aws elbv2 modify-target-group-attributes --target-group-arn <tg-arn> --attributes Key=deregistration_delay.timeout_seconds,Value=<seconds>`
- If requests dropped during deregistration: increase deregistration delay to allow in-flight requests to complete
- If application doesn't handle graceful shutdown: implement SIGTERM handler to finish in-flight requests
- If targets stuck in draining: wait for the timeout to expire — targets are force-deregistered after the delay

SHOULD:
- Set deregistration delay to match the maximum expected request processing time
- Implement health check failure on shutdown to speed up deregistration
- Configure the application to stop accepting new connections on SIGTERM

MAY:
- Enable connection draining on CLB: `aws elb modify-load-balancer-attributes --load-balancer-name <lb-name> --load-balancer-attributes "{\"ConnectionDraining\":{\"Enabled\":true,\"Timeout\":300}}"`
- Implement pre-stop hooks in Kubernetes for EKS-based targets
- Set up monitoring for deregistration duration

## Common Issues

- symptoms: "Deployments take too long due to connection draining"
  diagnosis: "Deregistration delay is set too high (e.g., 300 seconds) for the application's needs."
  resolution: "Reduce deregistration delay to match actual request processing time (e.g., 30-60 seconds)."

- symptoms: "Users see errors during deployments"
  diagnosis: "Deregistration delay is too short — in-flight requests are terminated."
  resolution: "Increase deregistration delay. Implement graceful shutdown in the application."

- symptoms: "WebSocket connections dropped during target deregistration"
  diagnosis: "WebSocket connections are long-lived and exceed the deregistration delay."
  resolution: "Increase deregistration delay or implement client-side reconnection logic."

- symptoms: "Target stays in draining state indefinitely"
  diagnosis: "This should not happen — targets are force-deregistered after the delay. Check if the target was re-registered."
  resolution: "Verify the target is not being re-registered by Auto Scaling. Check the actual deregistration delay value."

## Output Format

```yaml
root_cause: "Connection draining issue — <specific_cause>"
evidence:
  - type: draining_config
    content: "<deregistration delay settings>"
  - type: target_state
    content: "<targets in draining state>"
severity: MEDIUM
mitigation:
  immediate: "Adjust deregistration delay to match application requirements"
  long_term: "Implement graceful shutdown and proper deployment strategies"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get): GREEN — read-only"
  - "Phase 2 deregistration delay changes: YELLOW — affects in-flight request handling"
  - "Phase 2 application shutdown changes: YELLOW — affects application behavior"

## Escalation Conditions

escalation_conditions:
  - "Connection draining is causing production request failures"
  - "Deployment pipeline is blocked by draining targets"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Long-lived connections (WebSocket, gRPC) require special handling"
  - "Issue affects multiple target groups simultaneously"

## Data Sensitivity

data_sensitivity:
  - command: "describe-target-group-attributes"
    sensitivity: LOW
    contains: "Target group configuration settings"
  - command: "describe-target-health"
    sensitivity: LOW
    contains: "Target IDs and draining state"
  - command: "ALB access logs during draining"
    sensitivity: MEDIUM
    contains: "Request details during deregistration period"

## Prohibited Actions

prohibited_actions:
  - "NEVER set deregistration delay to 0 in production without understanding that in-flight requests will be dropped"
  - "NEVER force-terminate targets during draining without confirming no critical requests are in-flight"
  - "NEVER disable connection draining on CLB without a rollback plan"
  - "NEVER assume all requests complete instantly — always account for long-running requests"

## Phase 3 — Rollback

- Before changing deregistration delay, document the current value
- If reducing the delay causes dropped requests, increase it back
- If increasing the delay slows deployments, find a balanced value
- If application shutdown changes cause issues, revert the application code
- Monitor error rates during the next deployment after any changes
