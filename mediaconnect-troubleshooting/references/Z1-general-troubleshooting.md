---
title: "Z1 — General Troubleshooting"
description: "General MediaConnect troubleshooting for issues not covered by specific runbooks"
status: active
severity: MEDIUM
triggers: ["MediaConnect error", "flow issue", "general problem", "unknown error", "help with MediaConnect"]
owner: devops-agent
objective: "Systematically diagnose and resolve general MediaConnect issues"
context: "This catch-all runbook covers general troubleshooting steps when the specific issue category is unclear. Start here and escalate to specific runbooks as the issue is identified."
---

## Phase 1 — Triage

MUST:
- List all flows and check status: `aws mediaconnect list-flows --query 'Flows[*].{Name:Name,FlowArn:FlowArn,Status:Status,AvailabilityZone:AvailabilityZone}'`
- Describe the affected flow: `aws mediaconnect describe-flow --flow-arn <flow-arn>`
- Check key CloudWatch metrics: `aws cloudwatch get-metric-statistics --namespace AWS/MediaConnect --metric-name SourceBitRate --dimensions Name=FlowARN,Value=<flow-arn> --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Average`

SHOULD:
- Check CloudTrail for recent MediaConnect API errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=mediaconnect.amazonaws.com --max-results 20`
- Verify IAM permissions for the caller
- Check AWS Health Dashboard for service events

MAY:
- Review flow tags for environment and ownership information
- Check service quotas: `aws service-quotas list-service-quotas --service-code mediaconnect`
- Review recent changes in CloudTrail

## Phase 2 — Remediate

MUST:
- Identify the specific issue category and escalate to the appropriate runbook:
  - Flow creation → A1-flow-creation-failures
  - Flow configuration → A2-flow-configuration
  - Source connectivity → B1-source-connectivity
  - Source failover → B2-source-failover
  - Output delivery → C1-output-failures
  - Entitlements → C2-entitlement-issues
  - VPC interfaces → D1-vpc-interface-errors
  - Encryption → D2-encryption-issues
  - Transport stream → E1-transport-stream-errors
  - Bandwidth → E2-bandwidth-issues
  - Monitoring → F1-cloudwatch-monitoring
  - Maintenance → F2-maintenance-events
  - Multi-region → G1-multi-region-flows

SHOULD:
- Fix obvious issues (flow not started, permissions missing) before escalating
- Document the symptoms and initial findings

MAY:
- Restart the flow if the issue is unclear: stop then start
- Contact AWS Support with flow ARN and error details

## Safety Ratings

```
safety_ratings:
  - "list-flows: GREEN — read-only flow enumeration"
  - "describe-flow: GREEN — read-only flow inspection"
  - "cloudtrail lookup-events: GREEN — read-only audit log query"
  - "stop-flow / start-flow: RED — affects live video delivery"
```

## Escalation Conditions

- Issue cannot be categorized into any specific runbook
- Multiple simultaneous issues across different flows
- AWS service event affecting MediaConnect in the region
- Flow is in an unrecoverable state (not responding to start/stop)

## Data Sensitivity

- `aws mediaconnect list-flows`: LOW — returns flow metadata
- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs, encryption settings, VPC details
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities and actions
- `aws service-quotas list-service-quotas`: LOW — returns quota values

## Prohibited Actions

- NEVER delete a flow without confirming it is not in production use
- NEVER make multiple changes simultaneously — change one thing at a time
- NEVER ignore error messages — they contain diagnostic information
- NEVER assume the issue is with MediaConnect without checking source and network first

## Phase 3 — Rollback

1. If a flow was stopped for troubleshooting, restart: `aws mediaconnect start-flow --flow-arn <flow-arn>`
2. If any configuration was changed, revert to the documented previous state
3. If a flow was deleted, recreate from IaC templates or documented configuration
4. Verify all outputs and entitlements are functioning after any changes
5. Document the issue, resolution, and any configuration changes made

## Common Issues

- symptoms: "Flow is in ERROR state"
  diagnosis: "Flow encountered an unrecoverable error during operation."
  resolution: "Stop and restart the flow. If error persists, check source configuration and VPC interface settings."

- symptoms: "API calls return ThrottlingException"
  diagnosis: "API rate limit exceeded."
  resolution: "Implement exponential backoff in API calls. Reduce polling frequency. Use CloudWatch metrics instead of repeated describe-flow calls."

- symptoms: "Flow works in one region but not another"
  diagnosis: "Region-specific configuration differences (security groups, subnets, IAM roles)."
  resolution: "Compare flow configuration between regions. Verify all dependent resources exist in the target region."

## Output Format

```yaml
root_cause: "general — <specific_cause>"
evidence:
  - type: flow_status
    content: "<flow details and error information>"
severity: MEDIUM
mitigation:
  immediate: "Resolve identified issue using specific runbook"
  long_term: "Implement comprehensive monitoring and IaC for all flows"
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
  - "NEVER suggest disabling call recording without compliance review"
  - "NEVER suggest removing authentication"
  - "NEVER expose customer PII in logs"
