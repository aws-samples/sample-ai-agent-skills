---
title: "C1 — Route Analysis Errors"
description: "Diagnose route analysis failures and missing routes in core network segments"
status: active
severity: HIGH
triggers: ["route analysis error", "missing routes", "get-network-routes failed", "no routes found", "route not propagating"]
owner: devops-agent
objective: "Resolve route analysis errors and missing route issues in core network segments"
context: "Route analysis reflects the currently EXECUTED core network policy. Pending policy changes are not reflected until the change set is executed."
---

## Phase 1 — Triage

MUST:
- Get routes for the segment: `aws networkmanager get-network-routes --global-network-id <gn-id> --route-table-identifier Type=CORE_NETWORK_SEGMENT,CoreNetworkId=<cn-id>,SegmentName=<segment>`
- Verify the core network policy is executed: `aws networkmanager list-core-network-policy-versions --core-network-id <id>`
- List attachments in the segment: `aws networkmanager list-attachments --core-network-id <id> --attachment-type VPC`

SHOULD:
- Verify the attachment is in AVAILABLE state and assigned to the correct segment
- Check if the VPC CIDR is propagated to the segment route table
- Review the core network policy for segment-actions affecting this segment

MAY:
- Compare routes across multiple segments for consistency
- Check for overlapping CIDR blocks between attachments

## Phase 2 — Remediate

MUST:
- If routes missing due to unexecuted policy: execute the change set
- If attachment not in correct segment: verify tags match the attachment-policy conditions
- If CIDR overlap: resolve overlapping address ranges between VPCs

SHOULD:
- Verify route propagation after policy execution (allow up to 5 minutes)
- Check segment-actions for share or route-between rules affecting route visibility

MAY:
- Set up CloudWatch metrics for route count monitoring
- Document expected routes per segment for validation

## Common Issues

- symptoms: "get-network-routes returns empty for a segment with attachments"
  diagnosis: "Core network policy change was created but not executed."
  resolution: "Execute the pending change set: aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <version>."

- symptoms: "Routes visible but traffic not flowing between VPCs in same segment"
  diagnosis: "VPC route tables do not have routes pointing to the core network attachment."
  resolution: "Add routes in VPC route tables pointing to the core network attachment ENI or use route table propagation."

## Safety Ratings

```
safety_ratings:
  - "get-network-routes: GREEN — read-only route retrieval"
  - "list-core-network-policy-versions: GREEN — read-only version listing"
  - "execute-core-network-change-set: RED — activates policy changes affecting routing"
  - "list-attachments: GREEN — read-only enumeration"
```

## Escalation Conditions

- Routes missing after policy execution and 15+ minutes of propagation time
- Route analysis shows unexpected routes indicating possible policy misconfiguration
- Overlapping CIDRs between attachments causing ambiguous routing
- Route analysis API returning errors consistently across multiple segments

## Data Sensitivity

- `aws networkmanager get-network-routes`: MEDIUM — reveals network topology and CIDR allocations
- `aws networkmanager list-attachments`: LOW — returns attachment metadata
- `aws networkmanager get-core-network-policy`: MEDIUM — reveals segmentation strategy

## Prohibited Actions

- NEVER execute a policy change set solely to fix route analysis without reviewing the full change set
- NEVER delete attachments to resolve routing issues without confirming the attachment is the root cause
- NEVER modify VPC route tables in production without a rollback plan
- NEVER assume route analysis reflects real-time traffic — it reflects the executed policy

## Phase 3 — Rollback

1. If a policy execution caused route issues, create a rollback policy version reverting to the previous configuration
2. Execute the rollback policy: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <rollback-version>`
3. If VPC route tables were modified, revert them to previous state
4. Verify routes are restored: `aws networkmanager get-network-routes --global-network-id <gn-id> --route-table-identifier Type=CORE_NETWORK_SEGMENT,CoreNetworkId=<cn-id>,SegmentName=<segment>`
5. Confirm end-to-end connectivity from affected workloads

## Output Format

```yaml
root_cause: "route_analysis — <specific_cause>"
evidence:
  - type: route_state
    content: "<route details or absence>"
severity: HIGH
mitigation:
  immediate: "Fix routing configuration"
  long_term: "Implement route monitoring and alerting"
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
