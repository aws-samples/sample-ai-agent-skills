---
title: "C3 — SCP Conflicts"
description: "Diagnose issues where SCPs block legitimate operations or conflict with each other"
status: active
severity: HIGH
triggers:
  - "Access denied"
  - "SCP blocking"
  - "Cannot perform action"
  - "OrganizationException"
owner: devops-agent
objective: "Identify the conflicting SCP and resolve the access issue"
context: "SCPs define the maximum permissions for accounts. An explicit deny in any SCP overrides all allows. CT-managed SCPs and custom SCPs can conflict. SCPs do NOT apply to the management account."
---

## Phase 1 — Triage

MUST:
- Identify the denied action from the error message or CloudTrail
- List all SCPs on the affected account/OU: `aws organizations list-policies-for-target --target-id <ou-or-account-id> --filter SERVICE_CONTROL_POLICY`
- Read each SCP to find the deny statement: `aws organizations describe-policy --policy-id <id>`
- Determine if the blocking SCP is CT-managed or custom

SHOULD:
- Check the full SCP inheritance chain (root → parent OU → child OU → account)
- Use IAM Access Analyzer or policy simulator to test the effective permissions
- Check if the action is blocked by the region deny SCP

MAY:
- Check CloudTrail for the denied API call with error code
- Compare the SCP with the IAM policy to understand the effective permissions

## Phase 2 — Remediate

MUST:
- If CT-managed SCP: do NOT modify directly (causes drift). Use CT APIs to disable the control if appropriate
- If custom SCP: modify the SCP to allow the legitimate action
- If region deny: verify the region is governed or the service has a global exception

SHOULD:
- Add specific exceptions to custom SCPs rather than removing them entirely
- Document SCP changes and their justification
- Test SCP changes in a sandbox OU first

MAY:
- Consolidate overlapping SCPs to reduce complexity
- Implement SCP change management process

## Common Issues

- symptoms: "Action denied but IAM policy allows it"
  diagnosis: "An SCP is blocking the action at the Organization level."
  resolution: "Check all SCPs in the inheritance chain. Modify the custom SCP or disable the CT control."

- symptoms: "Region deny blocking a global service"
  diagnosis: "The service is not in the region deny SCP exceptions list."
  resolution: "Verify the service should be excepted. If so, the region deny SCP needs updating (via CT, not directly)."

- symptoms: "New SCP breaks existing workloads"
  diagnosis: "The SCP deny statement is too broad."
  resolution: "Add condition keys or resource exceptions to narrow the deny scope."

## Safety Ratings

```
safety_ratings:
  - "list-policies-for-target: GREEN — read-only SCP listing"
  - "describe-policy: GREEN — read-only SCP content"
  - "modify custom SCP to add exceptions: YELLOW — recoverable, SCP can be reverted"
  - "disable CT control (to remove CT-managed SCP): YELLOW — recoverable, control can be re-enabled"
  - "modify CT-managed SCP directly: RED — causes drift, breaks CT governance"
  - "remove SCP entirely: RED — removes security boundary for all accounts in the target"
```

## Escalation Conditions

- Fix requires modifying CT-managed SCPs
- Fix requires updating the landing zone version
- SCP conflict affects the region deny guardrail
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudTrail denied API call details (expose attempted actions)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If a custom SCP modification breaks workloads, revert the SCP to its previous version immediately
- Keep a backup of SCP content before any modifications
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If a CT control was disabled to resolve the conflict, document the justification and plan to re-enable with proper exceptions

## Output Format

```yaml
root_cause: "scp_conflict — <scp_name> blocking <action>"
evidence:
  - type: scp_policy
    content: "<deny statement details>"
  - type: cloudtrail_event
    content: "<denied API call>"
severity: HIGH
mitigation:
  immediate: "Modify custom SCP or disable CT control to unblock"
  long_term: "Implement SCP testing and change management process"
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
