---
title: "D1 — Sticky Sessions"
description: "Diagnose and resolve ELB sticky session (session affinity) issues"
status: active
severity: MEDIUM
triggers:
  - "sticky session"
  - "session affinity"
  - "AWSALB cookie"
  - "session persistence"
  - "stickiness"
owner: devops-agent
objective: "Resolve sticky session configuration issues and ensure proper session affinity behavior"
context: "ELB sticky sessions (session affinity) route requests from the same client to the same target. ALB supports duration-based stickiness (AWSALB cookie) and application-based stickiness (custom cookie). NLB supports source IP-based stickiness. Issues arise from misconfigured stickiness settings, cookie conflicts, uneven load distribution, or target group changes that break session affinity."
---

## Phase 1 — Triage

MUST:
- Check stickiness configuration: `aws elbv2 describe-target-group-attributes --target-group-arn <tg-arn> --query 'Attributes[?starts_with(Key, \`stickiness\`)]'`
- Verify stickiness is enabled: check `stickiness.enabled` attribute
- Check the stickiness type: `stickiness.type` (lb_cookie for duration-based, app_cookie for application-based)
- Verify target health — unhealthy targets break stickiness: `aws elbv2 describe-target-health --target-group-arn <tg-arn>`
- Check if the client is sending cookies: inspect request headers for AWSALB or custom cookie

SHOULD:
- Verify the stickiness duration: `stickiness.lb_cookie.duration_seconds`
- Check if the application sets its own session cookie (for app-based stickiness)
- Review load distribution across targets in CloudWatch: check RequestCountPerTarget metric
- Verify the listener rules route to the correct target group with stickiness

MAY:
- Check if cross-zone load balancing affects stickiness behavior
- Review if WebSocket connections are affected by stickiness settings
- Test stickiness with curl: `curl -v -c cookies.txt -b cookies.txt http://<lb-dns>/`

## Phase 2 — Remediate

MUST:
- If stickiness not enabled: enable it: `aws elbv2 modify-target-group-attributes --target-group-arn <tg-arn> --attributes Key=stickiness.enabled,Value=true Key=stickiness.type,Value=lb_cookie Key=stickiness.lb_cookie.duration_seconds,Value=<seconds>`
- If app-based stickiness cookie name wrong: update it: `aws elbv2 modify-target-group-attributes --target-group-arn <tg-arn> --attributes Key=stickiness.app_cookie.cookie_name,Value=<cookie-name>`
- If targets are unhealthy breaking stickiness: fix target health (see A1 runbook)
- If stickiness causes uneven load: adjust duration or consider disabling for stateless applications

SHOULD:
- Use application-based stickiness when the application manages its own sessions
- Set appropriate stickiness duration matching the session timeout
- Monitor load distribution after enabling stickiness

MAY:
- Implement server-side session storage (Redis, DynamoDB) to eliminate stickiness dependency
- Configure stickiness per target group for different application tiers
- Document the stickiness requirements and configuration

## Common Issues

- symptoms: "Sessions not sticky — requests go to different targets"
  diagnosis: "Stickiness not enabled, or client not sending cookies."
  resolution: "Enable stickiness on the target group. Verify the client accepts and sends cookies."

- symptoms: "All traffic goes to one target"
  diagnosis: "Stickiness duration too long or single client generating most traffic."
  resolution: "Reduce stickiness duration. Check if a load testing tool is using a single session."

- symptoms: "Session lost after target becomes unhealthy"
  diagnosis: "When a sticky target becomes unhealthy, ALB routes to a different target."
  resolution: "This is expected behavior. Implement server-side session storage for resilience."

- symptoms: "App-based stickiness not working"
  diagnosis: "Application cookie name doesn't match the configured cookie name, or the application isn't setting the cookie."
  resolution: "Verify the cookie name matches exactly. Ensure the application sets the cookie in responses."

## Output Format

```yaml
root_cause: "Sticky session issue — <specific_cause>"
evidence:
  - type: stickiness_config
    content: "<target group stickiness attributes>"
  - type: load_distribution
    content: "<RequestCountPerTarget metrics>"
severity: MEDIUM
mitigation:
  immediate: "Fix stickiness configuration or target health"
  long_term: "Implement server-side session storage for resilience"
```

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get): GREEN — read-only"
  - "Phase 2 stickiness configuration changes: YELLOW — affects traffic routing behavior"
  - "Phase 2 disabling stickiness: YELLOW — may break user sessions"

## Escalation Conditions

escalation_conditions:
  - "Stickiness changes affect active user sessions in production"
  - "Load imbalance caused by stickiness is causing target overload"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Application architecture requires stickiness but it cannot be maintained"
  - "Session loss is causing data corruption or transaction failures"

## Data Sensitivity

data_sensitivity:
  - command: "describe-target-group-attributes"
    sensitivity: LOW
    contains: "Target group configuration settings"
  - command: "cookie inspection"
    sensitivity: MEDIUM
    contains: "Session cookies that may contain session identifiers"
  - command: "CloudWatch metrics"
    sensitivity: LOW
    contains: "Aggregated request count metrics"

## Prohibited Actions

prohibited_actions:
  - "NEVER disable stickiness on a production target group without coordinating with the application team"
  - "NEVER set stickiness duration to extremely long values (days) without understanding memory impact"
  - "NEVER rely solely on stickiness for session management — implement server-side sessions"
  - "NEVER modify stickiness during peak traffic without a rollback plan"

## Phase 3 — Rollback

- Before changing stickiness settings, document current configuration
- If stickiness changes break user sessions, revert to previous settings immediately
- If load distribution becomes uneven, adjust duration or disable stickiness
- If app-based cookie name was changed, coordinate with the application team to revert
- Monitor session-related errors and load distribution after any changes
