---
title: "G1 — Policy Version Management"
description: "Diagnose core network policy version management, rollback, and lifecycle issues"
status: active
severity: HIGH
triggers: ["policy version error", "policy rollback failed", "wrong policy version active", "policy version conflict", "cannot revert policy"]
owner: devops-agent
objective: "Resolve core network policy version management and rollback issues"
context: "Core network policy versions are immutable. Only one version can be LIVE at a time. Rollback requires creating a new version with the desired configuration."
---

## Phase 1 — Triage

MUST:
- List all policy versions: `aws networkmanager list-core-network-policy-versions --core-network-id <id>`
- Identify the current LIVE version: look for the version with `LatestVersionId` matching the executed version
- Get the LIVE policy: `aws networkmanager get-core-network-policy --core-network-id <id> --policy-version-id <live-version>`

SHOULD:
- Get the desired rollback target policy: `aws networkmanager get-core-network-policy --core-network-id <id> --policy-version-id <target-version>`
- Review the change set between current and target versions
- Check CloudTrail for recent ExecuteCoreNetworkChangeSet events

MAY:
- Compare multiple policy versions to identify when the issue was introduced
- Export policy versions for offline comparison

## Phase 2 — Remediate

MUST:
- To rollback: submit the desired policy as a new version: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://rollback-policy.json`
- Review the change set: `aws networkmanager get-core-network-change-set --core-network-id <id> --policy-version-id <new-version>`
- Execute the rollback: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <new-version>`

SHOULD:
- Verify the rollback completed successfully by checking core network state
- Confirm attachments and routes are in expected state after rollback

MAY:
- Implement policy version tagging for easier identification
- Set up automated policy backup before changes

## Common Issues

- symptoms: "Cannot revert to a previous policy version directly"
  diagnosis: "Policy versions are immutable. There is no revert API — you must create a new version."
  resolution: "Get the desired policy version content, submit it as a new version via put-core-network-policy, then execute."

- symptoms: "Policy execution stuck in progress"
  diagnosis: "Complex policy changes (many attachments, segments) take time to propagate."
  resolution: "Wait up to 30 minutes for complex changes. Check core network state for UPDATING status."

## Safety Ratings

```
safety_ratings:
  - "list-core-network-policy-versions: GREEN — read-only version listing"
  - "get-core-network-policy: GREEN — read-only policy retrieval"
  - "put-core-network-policy: YELLOW — creates new policy version"
  - "execute-core-network-change-set: RED — activates policy changes"
```

## Escalation Conditions

- Policy execution fails or gets stuck for more than 30 minutes
- Rollback policy causes unexpected attachment rejections
- Multiple policy versions created in rapid succession causing confusion
- Policy version management requires coordination across multiple teams

## Data Sensitivity

- `aws networkmanager list-core-network-policy-versions`: LOW — returns version metadata
- `aws networkmanager get-core-network-policy`: MEDIUM — reveals full network architecture
- `aws cloudtrail lookup-events`: MEDIUM — reveals who executed policy changes

## Prohibited Actions

- NEVER execute a policy change set without reviewing it first
- NEVER assume a previous policy version can be directly reverted — versions are immutable
- NEVER delete policy versions (they cannot be deleted — they are retained for audit)
- NEVER execute rapid successive policy changes without waiting for each to complete

## Phase 3 — Rollback

1. If the wrong policy was executed, get the correct policy version content
2. Submit it as a new version: `aws networkmanager put-core-network-policy --core-network-id <id> --policy-document file://correct-policy.json`
3. Review the change set carefully
4. Execute the correction: `aws networkmanager execute-core-network-change-set --core-network-id <id> --policy-version-id <correction-version>`
5. Verify core network state returns to AVAILABLE and routes are correct

## Output Format

```yaml
root_cause: "policy_version — <specific_cause>"
evidence:
  - type: version_state
    content: "<policy version details>"
severity: HIGH
mitigation:
  immediate: "Execute correct policy version"
  long_term: "Implement policy-as-code with version control and CI/CD"
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
