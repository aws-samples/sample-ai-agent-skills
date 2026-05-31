---
title: "A3 — Throttling and Rate Limits"
description: "Diagnose Bedrock throttling and rate limit issues"
status: active
severity: HIGH
triggers:
  - "ThrottlingException"
  - "rate limit exceeded"
  - "too many requests"
  - "throttled"
owner: devops-agent
objective: "Identify and resolve Bedrock throttling issues"
context: "Each model has per-account rate limits for tokens-per-minute (TPM) and requests-per-minute (RPM). Exceeding limits returns ThrottlingException. Limits vary by model, region, and account tier. Provisioned throughput provides dedicated capacity."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch Throttles metric: `aws cloudwatch get-metric-statistics --namespace AWS/Bedrock --metric-name ThrottledCount --dimensions Name=ModelId,Value=<model-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`
- Verify current request rate against model limits
- Check for Retry-After header in throttled responses
- Identify the specific limit being hit (TPM or RPM)

SHOULD:
- Check invocation patterns for burst traffic
- Review CloudWatch Invocations metric for request volume
- Check if multiple applications share the same account/model
- Verify the account's rate limit tier

MAY:
- Check for regional differences in rate limits
- Review application retry logic

## Phase 2 — Remediate

MUST:
- Implement exponential backoff with jitter for retries
- Respect the Retry-After header in throttled responses
- Distribute requests evenly (avoid bursts)

SHOULD:
- Request rate limit increases through AWS support
- Consider provisioned throughput for consistent high-volume workloads
- Implement request queuing to smooth traffic
- Use multiple regions for geographic distribution

MAY:
- Implement client-side rate limiting
- Create dashboards for throttling monitoring

## Common Issues

- symptoms: "ThrottlingException during peak hours"
  diagnosis: "Request rate exceeds model's RPM or TPM limit."
  resolution: "Implement backoff/retry. Request limit increase. Consider provisioned throughput."

- symptoms: "Intermittent throttling with low request volume"
  diagnosis: "Token-per-minute limit hit due to large prompts/responses."
  resolution: "Reduce prompt/response size. Distribute requests over time."

## Output Format

```yaml
root_cause: "throttling — <specific_cause>"
evidence:
  - type: throttle_metrics
    content: "<CloudWatch throttle counts>"
  - type: request_rate
    content: "<current request rate vs limits>"
severity: HIGH
mitigation:
  immediate: "Implement retry with backoff"
  long_term: "Request limit increases or use provisioned throughput"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Implement exponential backoff with jitter for retries | GREEN |
| Respect the Retry-After header in throttled responses | GREEN |
| Distribute requests evenly (avoid bursts) | GREEN |
| Request rate limit increases through AWS support | GREEN |
| Consider provisioned throughput for consistent high-volume workloads | YELLOW |
| Implement request queuing to smooth traffic | GREEN |
| Use multiple regions for geographic distribution | YELLOW |
| Implement client-side rate limiting | GREEN |
| Create dashboards for throttling monitoring | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Throttling metrics may reveal usage patterns and workload characteristics

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If provisioned throughput was created, delete it to stop charges: `aws bedrock delete-provisioned-model-throughput --provisioned-model-id <id>`
2. If rate limit increases were applied, contact AWS support to revert if needed
3. If client-side retry logic was changed, revert application code to previous retry configuration
4. If multi-region distribution was set up, remove secondary region configurations if not needed
5. Verify rollback by confirming throttling metrics return to baseline

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
  - "NEVER suggest removing guardrails from production models"
  - "NEVER suggest disabling content filtering"
  - "NEVER suggest overly broad model access permissions"
