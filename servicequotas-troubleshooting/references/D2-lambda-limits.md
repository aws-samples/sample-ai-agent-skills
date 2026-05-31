---
title: "D2 — Lambda Limits"
description: "Troubleshoot common Lambda quota issues"
status: active
severity: HIGH
triggers:
  - "Lambda throttled"
  - "TooManyRequestsException"
  - "concurrency limit"
  - "Lambda limit"
owner: devops-agent
objective: "Resolve Lambda quota issues causing throttling or failures"
context: "Lambda concurrent execution quota is account-level per region (default 1,000). Reserved concurrency for one function reduces available pool. Other limits include function size, timeout, memory, and layers."
---

## Phase 1 — Triage

MUST:
- Check account concurrency limit: `aws lambda get-account-settings`
- Check concurrent execution quota: `aws service-quotas get-service-quota --service-code lambda --quota-code L-B99A9384`
- Check current concurrency usage: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name ConcurrentExecutions --start-time <start> --end-time <end> --period 60 --statistics Maximum`
- Check for throttled invocations: `aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles --dimensions Name=FunctionName,Value=<function> --start-time <start> --end-time <end> --period 300 --statistics Sum`

SHOULD:
- Check reserved concurrency per function: `aws lambda get-function-concurrency --function-name <function>`
- Check provisioned concurrency: `aws lambda list-provisioned-concurrency-configs --function-name <function>`
- Identify functions consuming the most concurrency

MAY:
- Check function code size limits
- Review function timeout and memory settings
- Check layer limits

## Phase 2 — Remediate

MUST:
- Request concurrency increase: `aws service-quotas request-service-quota-increase --service-code lambda --quota-code L-B99A9384 --desired-value <value>`
- Set reserved concurrency for critical functions: `aws lambda put-function-concurrency --function-name <function> --reserved-concurrent-executions <value>`

SHOULD:
- Use provisioned concurrency for latency-sensitive functions
- Optimize function duration to reduce concurrent execution needs
- Implement retry logic with exponential backoff in callers

MAY:
- Distribute functions across regions
- Use SQS to buffer invocations and control concurrency

## Common Issues

- symptoms: "TooManyRequestsException on Lambda invoke"
  diagnosis: "Account concurrent execution limit reached."
  resolution: "Request concurrency increase. Reserve concurrency for critical functions."

- symptoms: "One function throttled but others work"
  diagnosis: "Reserved concurrency set too low on the function."
  resolution: "Increase reserved concurrency or remove reservation to use shared pool."

- symptoms: "Function deployment fails — code too large"
  diagnosis: "Function package exceeds size limit (50MB zipped, 250MB unzipped)."
  resolution: "Use Lambda layers. Optimize dependencies. Use container images (10GB limit)."

## Output Format

```yaml
root_cause: "lambda_limits — <specific_cause>"
evidence:
  - type: concurrency_quota
    content: "<concurrent execution quota and usage>"
  - type: throttle_metrics
    content: "<throttle count>"
severity: HIGH
mitigation:
  immediate: "Request concurrency increase or optimize functions"
  long_term: "Implement concurrency management strategy"
```


## Safety Ratings

| Phase 2 Action | Rating | Rationale |
|---|---|---|
| Investigate and collect diagnostic data | GREEN | Read-only analysis; no state change |
| Modify configuration settings | YELLOW | Configuration change; reversible with backup |
| Restart or reboot components | YELLOW | Service interruption; recoverable |
| Delete or remove resources | RED | Potentially destructive; requires confirmation and backup |

## Escalation Conditions

- Production environment impact detected or imminent
- Multiple failed remediation attempts (>2 cycles without resolution)
- Security-related root cause identified (unauthorized access, data exposure)
- Issue requires cross-service or cross-account coordination
- Customer-facing service degradation exceeding SLA thresholds
- Regulatory or compliance implications discovered during investigation

## Data Sensitivity

- Quota utilization reveals infrastructure scale and capacity
- Service usage patterns may indicate business activity levels
- Organization quota templates reveal multi-account structure
- Classification: INTERNAL - operational capacity data

## Prohibited Actions

- NEVER reduce service quotas below current utilization levels
- NEVER delete CloudWatch alarms monitoring quota utilization
- NEVER modify organization-level quota templates without member account review
- NEVER cancel pending quota increase requests without requester approval

## Phase 3 -- Rollback

MUST:
- Document all changes made during Phase 2 remediation
- Verify rollback procedures exist before applying any fix
- Confirm service health after rollback using the same diagnostic steps from Phase 1
- Notify stakeholders if rollback is initiated in production

SHOULD:
- Maintain a timeline of all actions taken for post-incident review
- Capture before/after configuration snapshots
- Test rollback in non-production environment when possible
- Update runbook with lessons learned after incident resolution

MAY:
- Automate rollback procedures for frequently occurring issues
- Create CloudWatch alarms to detect regression after rollback

## Safety Ratings

safety_ratings:
  - "Phase 1 triage commands (describe/get/list): GREEN — read-only"
  - "Phase 2 configuration changes: YELLOW — state-changing but recoverable"
  - "Phase 2 resource deletion or security changes: RED — destructive or irreversible"

## Escalation Conditions

escalation_conditions:
  - "Remediation requires modifying IAM policies in a production account"
  - "Remediation requires disabling a security control even temporarily"
  - "Root cause cannot be identified after 3 hypothesis pivots"
  - "Blast radius affects more than one account or region"
  - "Issue involves potential data loss or exposure"

## Data Sensitivity

data_sensitivity:
  - command: "describe-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "list-* commands"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
