---
title: "C2 — Routing Policy Issues"
description: "Diagnose routing policy misconfiguration including segment-actions and attachment-policies"
status: active
severity: HIGH
triggers: ["routing policy error", "segment-actions not working", "attachment-policy mismatch", "traffic not routed correctly"]
owner: devops-agent
objective: "Resolve routing policy misconfigurations in core network policies"
context: "Routing behavior is controlled by the core network policy's segment-actions (share, route-between) and attachment-policies sections."
---

## Phase 1 — Triage

MUST:
- Get the active core network policy: `aws networkmanager get-core-network-policy --core-network-id <id>`
- Identify the LIVE policy version: `aws networkmanager list-core-network-policy-versions --core-network-id <id>`
- Check segment-actions and attachment-policies sections in the policy

SHOULD:
- Verify segment names referenced in segment-actions exist in the segments section
- Check if route-between or share actions are configured for the affected segments
- Review attachment-policies conditions for tag matching logic

MAY:
- Compare the LIVE policy with the most recent LATEST policy version
- Review CloudTrail for recent PutCoreNetworkPolicy events

## Phase 2 — Remediate

MUST:
- Fix segment-actions to enable required cross-segment routing
- Ensure attachment-policies conditions correctly match resource tags
- Submit corrected policy: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://corrected-policy.json`

SHOULD:
- Review the change set before executing: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <version>`
- Execute the corrected policy: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <version>`

MAY:
- Implement policy validation in CI/CD pipeline
- Create policy templates for common routing patterns

## Common Issues

- symptoms: "Traffic between two segments is blocked despite both having attachments"
  diagnosis: "No segment-actions (share or route-between) configured between the two segments."
  resolution: "Add a segment-actions entry with mode share or route-between for the affected segments."

- symptoms: "Attachment assigned to wrong segment"
  diagnosis: "Attachment-policy conditions match a different segment than intended."
  resolution: "Update resource tags or attachment-policy conditions to ensure correct segment assignment."

## Safety Ratings

```
safety_ratings:
  - "get-core-network-policy: GREEN — read-only policy retrieval"
  - "put-core-network-policy: YELLOW — creates new policy version, does not activate"
  - "execute-core-network-change-set: RED — activates routing changes across all segments"
  - "get-core-network-change-set: GREEN — read-only change preview"
```

## Escalation Conditions

- Routing policy change causes unintended cross-segment traffic in production
- Policy validation passes but execution causes attachment rejections
- Complex multi-segment routing requirements exceed policy capabilities
- Routing policy changes affect compliance-sensitive network segmentation

## Data Sensitivity

- `aws networkmanager get-core-network-policy`: MEDIUM — reveals full network segmentation and routing strategy
- `aws networkmanager get-core-network-change-set`: MEDIUM — reveals planned routing changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals who modified policies and when

## Prohibited Actions

- NEVER execute routing policy changes in production without reviewing the change set
- NEVER remove segment isolation (share/route-between) without security team approval
- NEVER modify attachment-policies that affect compliance-sensitive segments without change management
- NEVER assume segment-actions are bidirectional — verify the policy explicitly

## Phase 3 — Rollback

1. Create a rollback policy version with the previous routing configuration
2. Submit the rollback: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://rollback-policy.json`
3. Review the rollback change set: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <rollback-version>`
4. Execute the rollback: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <rollback-version>`
5. Verify segment isolation and routing are restored to expected state

## Output Format

```yaml
root_cause: "routing_policy — <specific_cause>"
evidence:
  - type: policy_config
    content: "<segment-actions or attachment-policies details>"
severity: HIGH
mitigation:
  immediate: "Fix routing policy configuration"
  long_term: "Implement policy-as-code with automated validation"
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
