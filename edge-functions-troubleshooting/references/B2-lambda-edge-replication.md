---
title: "B2 — Lambda@Edge Replication"
description: "Diagnose Lambda@Edge replication delays and failures"
status: active
severity: MEDIUM
triggers: ["replication slow", "function not replicated", "old version still running", "inconsistent behavior across regions"]
owner: devops-agent
objective: "Resolve Lambda@Edge replication issues across edge locations"
context: "Lambda@Edge functions are replicated from us-east-1 to regional edge caches. Replication typically takes 5-15 minutes but can take longer."
---

## Phase 1 — Triage

MUST:
- Check distribution status: `aws cloudfront get-distribution --id <distribution-id> --query 'Distribution.Status'`
- Verify the function version in the distribution config: `aws cloudfront get-distribution-config --id <distribution-id>`
- Check for replication errors in CloudWatch: `aws cloudwatch get-metric-statistics --namespace AWS/CloudFront --metric-name FunctionExecutionErrors --dimensions Name=DistributionId,Value=<dist-id> --start-time <start> --end-time <end> --period 60 --statistics Sum`

SHOULD:
- Wait at least 15 minutes after deployment before investigating
- Check if the distribution update completed (Status: Deployed)
- Test from multiple geographic locations to identify regional replication gaps

MAY:
- Check CloudFront distribution events in CloudTrail
- Monitor error rates across different edge locations

## Phase 2 — Remediate

MUST:
- Wait for distribution status to reach 'Deployed' before expecting full replication
- Verify the correct function version is in the distribution configuration
- If stuck in 'InProgress' for over 30 minutes, contact AWS Support

SHOULD:
- Implement health checks that verify function version at edge locations
- Use CloudFront real-time logs to monitor function execution across regions

MAY:
- Implement gradual rollout by updating cache behaviors one at a time
- Use CloudFront continuous deployment for safer updates

## Common Issues

- symptoms: "Some users see old behavior, others see new behavior"
  diagnosis: "Replication is in progress — different edge locations have different function versions."
  resolution: "Wait for distribution status to reach 'Deployed'. This typically takes 5-15 minutes."

- symptoms: "Distribution stuck in 'InProgress' for over 30 minutes"
  diagnosis: "Replication failure or service issue."
  resolution: "Check AWS Health Dashboard. If no service issues, contact AWS Support."

## Safety Ratings

```
safety_ratings:
  - "cloudfront get-distribution: GREEN — read-only"
  - "cloudfront get-distribution-config: GREEN — read-only"
  - "cloudwatch get-metric-statistics: GREEN — read-only"
  - "cloudfront update-distribution: RED — modifies CDN behavior"
```

## Escalation Conditions

- Distribution stuck in 'InProgress' for more than 30 minutes
- Replication causes errors at specific edge locations
- Inconsistent behavior persists after distribution reaches 'Deployed' status
- Business-critical traffic is affected by replication delays

## Data Sensitivity

- `aws cloudfront get-distribution`: MEDIUM — reveals distribution configuration
- `aws cloudfront get-distribution-config`: MEDIUM — reveals origin and behavior settings
- `aws cloudwatch get-metric-statistics`: LOW — aggregated metrics
- Geographic testing results: LOW — latency and response data

## Prohibited Actions

- NEVER force a distribution update while another update is in progress
- NEVER assume replication is complete before distribution status is 'Deployed'
- NEVER roll back during active replication — wait for completion first
- NEVER delete a Lambda@Edge function version while it's being replicated

## Phase 3 — Rollback

1. If the new function version causes issues, update distribution to previous version
2. Wait for the rollback distribution update to reach 'Deployed' status
3. Verify consistent behavior across geographic locations after rollback
4. Monitor error rates for 30 minutes after rollback completes
5. Document the replication issue and timeline

## Output Format

```yaml
root_cause: "replication — <specific_cause>"
evidence:
  - type: replication_status
    content: "<distribution status and timing details>"
severity: MEDIUM
mitigation:
  immediate: "Wait for replication or rollback"
  long_term: "Implement continuous deployment with automated health checks"
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
