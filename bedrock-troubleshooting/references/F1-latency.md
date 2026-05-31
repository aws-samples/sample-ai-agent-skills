---
title: "F1 — Latency Issues"
description: "Diagnose Bedrock model invocation latency problems"
status: active
severity: MEDIUM
triggers:
  - "high latency"
  - "slow response"
  - "invocation timeout"
  - "response time"
owner: devops-agent
objective: "Identify and resolve Bedrock latency issues"
context: "Latency depends on model size, input/output token count, concurrent requests, and whether using on-demand or provisioned throughput. First-token latency and total latency are different metrics. Streaming reduces perceived latency."
---

## Phase 1 — Triage

MUST:
- Check CloudWatch latency metrics: `aws cloudwatch get-metric-statistics --namespace AWS/Bedrock --metric-name InvocationLatency --dimensions Name=ModelId,Value=<model-id> --start-time <start> --end-time <end> --period 300 --statistics Average p99`
- Verify input and output token counts
- Check for throttling (throttled requests have higher latency)
- Compare latency across different times of day

SHOULD:
- Check first-token latency vs total latency
- Verify if using streaming (reduces perceived latency)
- Compare latency across models
- Check for network latency (VPC endpoint vs public)

MAY:
- Profile end-to-end application latency
- Check for client-side processing delays

## Phase 2 — Remediate

MUST:
- Reduce input/output token counts where possible
- Use streaming for better perceived latency
- Implement appropriate timeouts in client code

SHOULD:
- Use provisioned throughput for consistent latency
- Choose smaller models for latency-sensitive use cases
- Implement caching for repeated queries
- Use the closest region for lower network latency

MAY:
- Implement latency monitoring and alerting
- Create latency budgets for different use cases

## Common Issues

- symptoms: "High latency on large prompts"
  diagnosis: "Large input token count increases processing time."
  resolution: "Reduce prompt size. Summarize context. Use streaming."

- symptoms: "Inconsistent latency"
  diagnosis: "On-demand throughput has variable latency under load."
  resolution: "Use provisioned throughput for consistent latency."

## Output Format

```yaml
root_cause: "latency — <specific_cause>"
evidence:
  - type: latency_metrics
    content: "<CloudWatch latency statistics>"
  - type: token_counts
    content: "<input/output token analysis>"
severity: MEDIUM
mitigation:
  immediate: "Optimize token counts and use streaming"
  long_term: "Implement latency monitoring and consider provisioned throughput"
```


## Safety Ratings

| Phase 2 Action | safety_rating |
|---|---|
| Reduce input/output token counts where possible | GREEN |
| Use streaming for better perceived latency | GREEN |
| Implement appropriate timeouts in client code | GREEN |
| Use provisioned throughput for consistent latency | YELLOW |
| Choose smaller models for latency-sensitive use cases | YELLOW |
| Implement caching for repeated queries | GREEN |
| Use the closest region for lower network latency | YELLOW |
| Implement latency monitoring and alerting | GREEN |
| Create latency budgets for different use cases | GREEN |

## Escalation Conditions

- Fix involves modifying guardrails on production model
- Knowledge base contains sensitive data

## Data Sensitivity

- **HIGH**: model invocation data, knowledge base content
- Latency metrics and token counts may reveal workload characteristics and usage patterns

## Prohibited Actions

- NEVER suggest removing guardrails to fix content filtering
- NEVER suggest using production data for fine-tuning without approval

## Phase 3 — Rollback

1. If the model was changed for latency reasons, revert to the previous model ID in application configuration
2. If provisioned throughput was created, delete it if no longer needed
3. If caching was implemented, disable the cache layer if it causes stale responses
4. If client timeouts were changed, restore previous timeout values
5. Verify rollback by measuring latency metrics and confirming acceptable performance

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
