---
title: "E2 — Nested OU Issues"
description: "Diagnose issues with nested organizational unit structures in Control Tower"
status: active
severity: MEDIUM
triggers:
  - "Nested OU"
  - "Child OU not governed"
  - "Controls not applied to nested OU"
owner: devops-agent
objective: "Resolve governance gaps in nested OU structures"
context: "Control Tower supports nested OUs up to 5 levels deep. Each OU must be individually registered. Controls must be enabled on each OU separately — they do NOT inherit from parent OUs. This is a common source of governance gaps."
---

## Phase 1 — Triage

MUST:
- Map the full OU hierarchy: `aws organizations list-organizational-units-for-parent --parent-id <parent-id>`
- Check which OUs are registered with CT
- Check which controls are enabled on each OU level
- Identify governance gaps (registered parent but unregistered child)

SHOULD:
- Verify StackSet deployments target all registered OUs
- Check if accounts in nested OUs are governed
- Compare control coverage across OU levels

MAY:
- Visualize the OU hierarchy with registration and control status
- Check if SCP inheritance (Organizations-level) compensates for CT control gaps

## Phase 2 — Remediate

MUST:
- Register all child OUs that need governance
- Enable required controls on each registered OU individually
- Verify accounts in nested OUs are governed after registration

SHOULD:
- Document the OU hierarchy and governance requirements
- Create a control enablement matrix (which controls on which OUs)
- Standardize control sets across OU levels

MAY:
- Automate OU registration and control enablement for new OUs
- Simplify the OU structure if nesting is causing operational complexity

## Common Issues

- symptoms: "Controls enabled on parent OU but child OU accounts are non-compliant"
  diagnosis: "CT controls do not inherit to child OUs."
  resolution: "Enable the controls on the child OU individually."

- symptoms: "Child OU not showing in CT console"
  diagnosis: "The child OU is not registered with Control Tower."
  resolution: "Register the child OU with Control Tower."

- symptoms: "Accounts in nested OU not receiving baseline"
  diagnosis: "The nested OU is not registered, so StackSets don't target it."
  resolution: "Register the nested OU to deploy baselines to its accounts."

## Safety Ratings

```
safety_ratings:
  - "list-organizational-units-for-parent: GREEN — read-only OU hierarchy"
  - "list-enabled-controls: GREEN — read-only control listing"
  - "register child OU: YELLOW — recoverable, OU can be deregistered"
  - "enable controls on nested OU: YELLOW — recoverable, controls can be disabled"
  - "simplify OU structure (move accounts, delete OUs): RED — can break governance for all affected accounts"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Nested OU structure exceeds 5 levels (CT limit)
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Control enablement matrix across OU hierarchy

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If a child OU registration fails, it does not affect the parent OU — investigate the child OU independently
- Document the full OU hierarchy and control enablement matrix before making structural changes
- If OU structure simplification is needed, move accounts one at a time and verify governance after each move

## Output Format

```yaml
root_cause: "nested_ou — <governance gap description>"
evidence:
  - type: ou_hierarchy
    content: "<OU structure with registration status>"
severity: MEDIUM
mitigation:
  immediate: "Register child OUs and enable controls"
  long_term: "Document OU governance matrix and automate control enablement"
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
  - "NEVER suggest disabling mandatory controls"
  - "NEVER suggest removing the audit account"
  - "NEVER modify landing zone without understanding blast radius"
