---
title: "G1 — Multi-Region Flows"
description: "Diagnose MediaConnect multi-region flow configuration and connectivity issues"
status: active
severity: HIGH
triggers: ["multi-region", "cross-region flow", "region failover", "entitlement cross-region", "gateway bridge"]
owner: devops-agent
objective: "Resolve multi-region flow delivery issues using entitlements or MediaConnect Gateway"
context: "MediaConnect flows are regional. Multi-region delivery requires separate flows connected via entitlements or MediaConnect Gateway bridges. There is no automatic cross-region replication."
---

## Phase 1 — Triage

MUST:
- List flows in source region: `aws mediaconnect list-flows --region <source-region> --query 'Flows[*].{Name:Name,FlowArn:FlowArn,Status:Status}'`
- List flows in destination region: `aws mediaconnect list-flows --region <dest-region> --query 'Flows[*].{Name:Name,FlowArn:FlowArn,Status:Status}'`
- Check entitlements on source flow: `aws mediaconnect describe-flow --flow-arn <source-flow-arn> --region <source-region> --query 'Flow.Entitlements'`

SHOULD:
- Verify subscriber flow in destination region uses the entitlement ARN as source
- Check both flows are ACTIVE
- Verify source flow has healthy bitrate

MAY:
- Check MediaConnect Gateway configuration if using gateway bridges: `aws mediaconnect list-gateways`
- Review cross-region network latency

## Phase 2 — Remediate

MUST:
- Create entitlement on source flow for the destination account: `aws mediaconnect grant-flow-entitlements --flow-arn <source-flow-arn> --region <source-region> --entitlements Subscribers=<dest-account-id>,Description="Cross-region delivery"`
- Create subscriber flow in destination region using entitlement: source configuration references the entitlement ARN
- Ensure both flows are started

SHOULD:
- Configure source failover on both source and destination flows
- Set up CloudWatch monitoring in both regions
- Document the cross-region flow topology

MAY:
- Use MediaConnect Gateway for on-premises to multi-region delivery
- Configure entitlement encryption for cross-region content protection
- Set up cross-region CloudWatch dashboards

## Safety Ratings

```
safety_ratings:
  - "list-flows: GREEN — read-only flow enumeration"
  - "describe-flow: GREEN — read-only flow inspection"
  - "grant-flow-entitlements: YELLOW — shares content with another account/region"
  - "create-flow (destination region): YELLOW — creates new flow resource with cost"
```

## Escalation Conditions

- Cross-region entitlement shows ENABLED but destination flow receives no content
- Latency between regions causes unacceptable delay for live content
- Gateway bridge connectivity fails between on-premises and cloud
- Multi-region failover does not activate when source region flow fails

## Data Sensitivity

- `aws mediaconnect list-flows`: LOW — returns flow metadata
- `aws mediaconnect describe-flow`: MEDIUM — contains source IPs and entitlement details
- `aws mediaconnect grant-flow-entitlements`: HIGH — grants cross-account content access
- `aws mediaconnect list-gateways`: MEDIUM — reveals gateway infrastructure topology

## Prohibited Actions

- NEVER assume a flow in one region can directly receive content from another region
- NEVER grant entitlements to unverified account IDs for cross-region sharing
- NEVER create cross-region flows without understanding data transfer cost implications
- NEVER rely on a single region for critical live content — implement multi-region redundancy

## Phase 3 — Rollback

1. If cross-region entitlement was granted incorrectly, revoke: `aws mediaconnect revoke-flow-entitlement --flow-arn <source-flow-arn> --region <source-region> --entitlement-arn <entitlement-arn>`
2. If destination flow was created in error, delete: `aws mediaconnect delete-flow --flow-arn <dest-flow-arn> --region <dest-region>`
3. If gateway bridge was misconfigured, update or remove the bridge configuration
4. Verify source flow is unaffected by the rollback
5. Document the multi-region topology change and update the architecture diagram

## Common Issues

- symptoms: "Destination region flow shows no source bitrate despite entitlement"
  diagnosis: "Source flow is not ACTIVE, entitlement is DISABLED, or subscriber account ID is wrong."
  resolution: "Verify source flow is ACTIVE with healthy bitrate. Check entitlement status is ENABLED and subscriber account matches."

- symptoms: "Cross-region latency is too high for live content"
  diagnosis: "Geographic distance between regions adds inherent latency."
  resolution: "Choose regions closer to each other. Use SRT protocol with appropriate latency settings. Consider edge locations."

- symptoms: "Gateway bridge shows disconnected status"
  diagnosis: "On-premises gateway instance cannot reach the cloud gateway endpoint."
  resolution: "Verify network connectivity from on-premises to the gateway endpoint. Check firewall rules and NAT configuration."

## Output Format

```yaml
root_cause: "multi_region — <specific_cause>"
evidence:
  - type: cross_region_config
    content: "<source and destination flow details, entitlement status>"
severity: HIGH
mitigation:
  immediate: "Fix cross-region connectivity"
  long_term: "Implement automated multi-region flow management with IaC"
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
