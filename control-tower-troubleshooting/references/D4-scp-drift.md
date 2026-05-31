---
title: "D4 — SCP Drift"
description: "Diagnose and resolve drift caused by modifications to CT-managed SCPs"
status: active
severity: CRITICAL
triggers:
  - "SCP modified"
  - "SCP detached"
  - "SCP deleted"
  - "Policy drift"
owner: devops-agent
objective: "Identify SCP drift and restore CT-managed SCPs to expected state"
context: "Control Tower manages specific SCPs for preventive controls. Modifying, detaching, or deleting these SCPs causes drift and breaks control enforcement. CT-managed SCPs have specific naming patterns."
---

## Phase 1 — Triage

MUST:
- List all SCPs and identify CT-managed ones: `aws organizations list-policies --filter SERVICE_CONTROL_POLICY`
- Check if CT-managed SCPs are still attached to the correct targets
- Compare SCP content with expected CT baseline
- Check CloudTrail for UpdatePolicy, DetachPolicy, or DeletePolicy events on CT SCPs

SHOULD:
- Identify which preventive controls are affected by the SCP change
- Check if the SCP modification was intentional or accidental
- Verify the SCP change hasn't broken workloads

MAY:
- Diff the current SCP content with the expected content
- Check if custom SCPs were accidentally attached to CT-managed targets

## Phase 2 — Remediate

MUST:
- Do NOT manually restore the SCP (this may cause further drift)
- Use CT re-register OU to restore CT-managed SCPs
- If SCP was deleted: re-register the OU to recreate it

SHOULD:
- Verify control enforcement after SCP restoration
- Document the SCP change that caused drift
- Implement monitoring for SCP modifications

MAY:
- Use AWS Config rules to detect SCP modifications
- Implement IAM policies to restrict SCP modifications (limited since management account is exempt)

## Common Issues

- symptoms: "Preventive control shows as enabled but not enforcing"
  diagnosis: "The underlying SCP was modified or detached."
  resolution: "Re-register the OU to restore the SCP."

- symptoms: "CT shows drift after SCP update"
  diagnosis: "A CT-managed SCP was directly modified."
  resolution: "Re-register the affected OU. Do not manually edit the SCP again."

## Safety Ratings

```
safety_ratings:
  - "list-policies: GREEN — read-only SCP listing"
  - "describe-policy: GREEN — read-only SCP content"
  - "re-register OU to restore SCPs: YELLOW — recoverable, resets CT baselines"
  - "manually restore SCP content: RED — can cause further drift if done incorrectly"
  - "modify CT-managed SCPs: RED — causes additional drift, breaks CT governance"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- CT-managed SCP was deleted (not just modified)
- Drift affects multiple OUs or accounts
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: CloudTrail SCP modification events (expose who changed what)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Do NOT manually edit CT-managed SCPs to fix drift — always use CT re-register OU
- Document the current SCP content and attachment state before any remediation
- If the SCP modification was intentional (e.g., to unblock a workload), document the business justification before restoring the original

## Output Format

```yaml
root_cause: "scp_drift — <scp_name> <modification_type>"
evidence:
  - type: organizations_policy
    content: "<SCP current vs expected state>"
  - type: cloudtrail_events
    content: "<SCP modification event>"
severity: CRITICAL
mitigation:
  immediate: "Re-register the OU to restore CT-managed SCPs"
  long_term: "Implement SCP modification monitoring and change management"
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
