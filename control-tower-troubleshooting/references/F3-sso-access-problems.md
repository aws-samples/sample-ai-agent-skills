---
title: "F3 — SSO Access Problems"
description: "Diagnose issues where users cannot access accounts through IAM Identity Center"
status: active
severity: HIGH
triggers:
  - "Cannot access account"
  - "SSO access denied"
  - "No accounts visible in portal"
  - "Session expired"
owner: devops-agent
objective: "Restore user access through IAM Identity Center"
context: "Access problems can stem from missing account assignments, expired sessions, MFA issues, permission set not provisioned, or identity source synchronization delays."
---

## Phase 1 — Triage

MUST:
- Verify the user exists in the identity source
- Check account assignments for the user/group: `aws sso-admin list-account-assignments --instance-arn <arn> --account-id <id> --permission-set-arn <ps-arn>`
- Verify the permission set is provisioned to the target account
- Check if the user's group membership is correct

SHOULD:
- Check MFA configuration and status
- Verify session duration hasn't expired
- Check if the identity source sync is current (for external IdP)

MAY:
- Check CloudTrail for authentication events
- Review the SSO access portal for error messages

## Phase 2 — Remediate

MUST:
- Create missing account assignments
- Provision permission sets to accounts if not provisioned
- Fix group membership if incorrect

SHOULD:
- Verify access after remediation
- Check if the issue affects other users
- Review MFA configuration

MAY:
- Force identity source sync for external IdP
- Clear browser cache/cookies for SSO portal issues

## Common Issues

- symptoms: "User sees no accounts in SSO portal"
  diagnosis: "No account assignments exist for the user or their groups."
  resolution: "Assign the user or their group to accounts with appropriate permission sets."

- symptoms: "User can see account but gets access denied"
  diagnosis: "Permission set is assigned but not provisioned, or the IAM role is missing."
  resolution: "Re-provision the permission set to the account."

- symptoms: "Access works for some users but not others"
  diagnosis: "Group membership or individual assignment differences."
  resolution: "Check group membership and account assignments for the affected user."

## Safety Ratings

```
safety_ratings:
  - "list-account-assignments: GREEN — read-only assignment listing"
  - "list-instances: GREEN — read-only SSO instance listing"
  - "create account assignment: YELLOW — recoverable, assignment can be removed"
  - "provision permission set: YELLOW — recoverable, redeploys IAM roles"
  - "force identity source sync: YELLOW — recoverable, triggers sync cycle"
```

## Escalation Conditions

- Fix requires updating the landing zone version
- Fix requires modifying CT-managed SCPs
- Access issue affects all users (identity source problem)
- SSO instance-level issue requires AWS Support
- Fix requires moving the Audit or Log Archive account

## Data Sensitivity

- HIGH: Organization account list (all account IDs and emails)
- HIGH: SSO user/group assignments (expose who has access to what)
- HIGH: Permission set policies (expose granted permissions)
- MEDIUM: OU structure, control enablement status
- MEDIUM: MFA configuration and session settings

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If account assignments were changed and break access, revert the assignments immediately
- Document current user/group assignments before making changes
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- If MFA configuration was changed to troubleshoot access, restore the original MFA settings after the issue is resolved

## Output Format

```yaml
root_cause: "sso_access — <specific cause>"
evidence:
  - type: sso_assignments
    content: "<assignment and provisioning status>"
severity: HIGH
mitigation:
  immediate: "Fix assignments or provisioning"
  long_term: "Implement access review process and monitoring"
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
