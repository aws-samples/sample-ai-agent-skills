---
title: "F1 — Throttling Issues"
description: "Diagnose MediaStore throttling and rate limiting issues"
status: active
severity: HIGH
triggers: ["throttling", "rate limited", "429 error", "too many requests", "slow mediastore"]
owner: devops-agent
objective: "Resolve MediaStore throttling issues and optimize request patterns"
context: "MediaStore has per-container transaction limits. Throttling returns 429 errors. Limits can be increased through AWS Support."
---

## Phase 1 — Triage

MUST:
- Check throttling metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaStore --metric-name ThrottleCount --dimensions Name=ContainerName,Value=<name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Check request count: `aws cloudwatch get-metric-statistics --namespace AWS/MediaStore --metric-name RequestCount --dimensions Name=ContainerName,Value=<name> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Identify the operation type being throttled (PutObject, GetObject, ListItems)

SHOULD:
- Check if throttling correlates with specific time periods (live event starts)
- Verify the request pattern (burst vs sustained)

MAY:
- Enable detailed metric policy for path-level throttling analysis
- Review application request patterns for optimization opportunities

## Phase 2 — Remediate

MUST:
- Implement exponential backoff and retry in the application
- Distribute requests across multiple containers if possible
- Request limit increase through AWS Support for sustained high-throughput needs

SHOULD:
- Use CloudFront to cache GET requests and reduce origin load
- Optimize request patterns (batch operations, reduce unnecessary list-items calls)

MAY:
- Implement client-side caching for frequently accessed objects
- Use multiple containers to distribute write load

## Safety Ratings

```
safety_ratings:
  - "cloudwatch get-metric-statistics: GREEN — read-only metric query"
  - "Implementing retry logic: GREEN — application-level change"
  - "Distributing across containers: YELLOW — architecture change"
  - "Requesting limit increase: GREEN — AWS Support request"
```

## Escalation Conditions

- Throttling causes live video ingestion failures (time-critical)
- Request rate exceeds the maximum limit increase available
- Throttling occurs at rates well below documented limits
- Multiple containers are simultaneously throttled

## Data Sensitivity

- Throttle metrics: LOW — numeric request counts
- Request patterns: LOW — timing and volume data
- Container names: LOW — resource identifiers
- AWS Support case details: MEDIUM — may contain architecture details

## Prohibited Actions

- NEVER ignore throttling errors — they indicate capacity issues
- NEVER retry throttled requests without backoff — it worsens the problem
- NEVER create excessive containers solely to avoid throttling without architecture review
- NEVER assume throttling limits are the same across all regions

## Phase 3 — Rollback

1. If application retry logic was changed, revert to previous configuration
2. If requests were redistributed across containers, consolidate if needed
3. If limit increase was applied, it can be reverted through AWS Support
4. Verify throttling has subsided after changes
5. Document the throttling pattern and resolution

## Common Issues

- symptoms: "429 Too Many Requests during live event start"
  diagnosis: "Burst of concurrent viewers exceeds the container's request rate limit."
  resolution: "Use CloudFront to cache GET requests. Request limit increase before events. Implement retry with backoff."

- symptoms: "Sustained throttling on PutObject during live ingestion"
  diagnosis: "Ingestion rate exceeds the container's write throughput limit."
  resolution: "Request write limit increase through AWS Support. Consider splitting streams across containers."

## Output Format

```yaml
root_cause: "throttling — <specific_cause>"
evidence:
  - type: throttle_metric
    content: "<metric details>"
severity: HIGH
mitigation:
  immediate: "Implement backoff and request limit increase"
  long_term: "Optimize architecture for throughput requirements"
```

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
