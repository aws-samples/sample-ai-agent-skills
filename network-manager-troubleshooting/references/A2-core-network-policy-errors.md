---
title: "A2 — Core Network Policy Errors"
description: "Diagnose core network policy validation and execution failures"
status: active
severity: CRITICAL
triggers: ["core network policy error", "policy validation failed", "policy execution failed", "put-core-network-policy error"]
owner: devops-agent
objective: "Resolve core network policy creation, validation, and execution issues"
context: "Core network policies define segments, attachment-policies, and segment-actions. Policies must be explicitly executed after creation."
---

## Phase 1 — Triage

MUST:
- Get current core network state: `aws networkmanager get-core-network --core-network-id <id>`
- List policy versions: `aws networkmanager list-core-network-policy-versions --core-network-id <id>`
- Get the current LIVE policy: `aws networkmanager get-core-network-policy --core-network-id <id> --policy-version-id <version>`

SHOULD:
- Review the change set for pending policy: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <version>`
- Check CloudTrail for PutCoreNetworkPolicy errors

MAY:
- Compare current LIVE policy with the failed policy version
- Validate the policy JSON structure offline before submitting

## Phase 2 — Remediate

MUST:
- Fix policy validation errors identified in the error response
- Verify all edge-locations use valid AWS Region codes (e.g., us-east-1)
- Ensure segment names are unique and do not contain special characters
- Verify attachment-policies conditions reference valid segment names

SHOULD:
- Review the change set before executing: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <version>`
- Execute the corrected policy: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <version>`

MAY:
- Use a JSON schema validator for the policy document
- Implement policy-as-code with version control

## Common Issues

- symptoms: "PutCoreNetworkPolicy returns ValidationException with edge-location error"
  diagnosis: "Invalid AWS Region code in edge-locations array."
  resolution: "Use valid Region codes (e.g., us-east-1, eu-west-1). Check supported Cloud WAN edge locations."

- symptoms: "Policy created but routes not updated"
  diagnosis: "Policy version was created but change set was not executed."
  resolution: "Execute the change set: aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <version>."

## Safety Ratings

```
safety_ratings:
  - "get-core-network-policy: GREEN — read-only policy retrieval"
  - "list-core-network-policy-versions: GREEN — read-only version listing"
  - "put-core-network-policy: YELLOW — creates new policy version but does not activate"
  - "execute-core-network-change-set: RED — activates policy changes affecting all attachments and routing"
```

## Escalation Conditions

- Policy execution fails with internal errors after fixing all validation issues
- Policy execution causes unexpected attachment rejections in production
- Core network stuck in UPDATING state for more than 30 minutes after policy execution
- Policy change affects cross-account attachments requiring coordination

## Data Sensitivity

- `aws networkmanager get-core-network-policy`: MEDIUM — reveals network architecture and segmentation strategy
- `aws networkmanager get-core-network-change-set`: MEDIUM — reveals planned network changes
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities

## Prohibited Actions

- NEVER execute a core network change set without reviewing it first
- NEVER delete a policy version that is currently LIVE
- NEVER modify segment names in a policy if attachments are associated with those segments
- NEVER execute policy changes in production without testing in a non-production core network first

## Phase 3 — Rollback

1. If a bad policy was executed, create a new policy version reverting to the previous configuration
2. Submit the rollback policy: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://rollback-policy.json`
3. Review the rollback change set: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <new-version>`
4. Execute the rollback: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <new-version>`
5. Verify attachments and routes are restored to expected state

## Output Format

```yaml
root_cause: "core_network_policy — <specific_cause>"
evidence:
  - type: policy_error
    content: "<validation or execution error details>"
severity: CRITICAL
mitigation:
  immediate: "Fix policy and re-execute"
  long_term: "Implement policy-as-code with CI/CD validation"
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
