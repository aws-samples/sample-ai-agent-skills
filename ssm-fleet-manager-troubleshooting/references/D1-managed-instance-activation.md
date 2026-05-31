---
title: "D1 — Managed Instance Activation"
description: "Diagnose managed instance activation creation and usage issues"
status: active
severity: HIGH
triggers: ["activation failed", "cannot create activation", "activation code not working"]
owner: devops-agent
objective: "Resolve managed instance activation issues for hybrid environments"
context: "Activations enable on-premises servers and non-EC2 VMs to register as managed instances (mi-* IDs). Each activation has a registration limit and expiration date."
---

## Phase 1 — Triage

MUST:
- List existing activations: `aws ssm describe-activations --query 'ActivationList[*].{Id:ActivationId,Expired:Expired,Limit:RegistrationLimit,Count:RegistrationsCount,Role:IamRole}'`
- Check if activation is expired or registration limit reached
- Verify the IAM role specified in the activation exists: `aws iam get-role --role-name <role-name>`

SHOULD:
- Check the managed instance tier: `aws ssm get-service-setting --setting-id arn:aws:ssm:<region>:<account-id>:servicesetting/ssm/managed-instance/activation-tier`
- Verify the IAM role has the SSM trust policy for ssm.amazonaws.com

MAY:
- Review CloudTrail for CreateActivation errors
- Check if the 1,000 standard-tier on-premises instance limit has been reached

## Phase 2 — Remediate

MUST:
- Create new activation if expired: `aws ssm create-activation --iam-role <role-name> --registration-limit <limit> --default-instance-name <name>`
- Ensure IAM role trust policy includes ssm.amazonaws.com as trusted service
- Verify the role has `AmazonSSMManagedInstanceCore` policy attached

SHOULD:
- Set appropriate registration limits and expiration dates
- Use descriptive default instance names for identification

MAY:
- Upgrade to advanced tier if approaching 1,000 on-premises instance limit
- Automate activation creation and distribution via configuration management

## Common Issues

- symptoms: "create-activation returns AccessDeniedException"
  diagnosis: "IAM permissions missing for ssm:CreateActivation or iam:PassRole."
  resolution: "Add ssm:CreateActivation and iam:PassRole permissions to the caller's IAM policy."

- symptoms: "Activation code used but instance not appearing"
  diagnosis: "IAM role trust policy missing ssm.amazonaws.com or network connectivity issue."
  resolution: "Update role trust policy. Verify on-premises server can reach SSM endpoints over HTTPS."

## Safety Ratings

```
safety_ratings:
  - "describe-activations: GREEN — read-only activation listing"
  - "create-activation: YELLOW — creates activation code for instance registration"
  - "get-service-setting: GREEN — read-only tier check"
  - "iam get-role: GREEN — read-only role inspection"
```

## Escalation Conditions

- Activation creation fails with service errors despite correct IAM configuration
- On-premises instances register but immediately show as Connection Lost
- Registration limit reached and business requires more hybrid instances
- Activation codes need to be distributed securely to remote sites

## Data Sensitivity

- `aws ssm describe-activations`: MEDIUM — reveals activation IDs and IAM roles
- `aws ssm create-activation`: HIGH — generates activation codes that grant SSM access
- `aws iam get-role`: MEDIUM — reveals IAM role configuration and trust policies
- Activation codes: HIGH — anyone with the code can register instances in your account

## Prohibited Actions

- NEVER share activation codes over unencrypted channels (email, chat)
- NEVER create activations with unlimited registration limits in production
- NEVER use the same activation code across multiple untrusted environments
- NEVER delete activations that have active registered instances without deregistering them first

## Phase 3 — Rollback

1. If an activation was created with incorrect settings, delete it: `aws ssm delete-activation --activation-id <id>`
2. If instances were registered with wrong activation, deregister them: `aws ssm deregister-managed-instance --instance-id <mi-id>`
3. If IAM role was modified incorrectly, restore the previous trust policy
4. Verify remaining activations and registered instances are correct
5. Document activation changes and notify the infrastructure team

## Output Format

```yaml
root_cause: "managed_instance_activation — <specific_cause>"
evidence:
  - type: activation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix activation issue"
  long_term: "Automate activation lifecycle management"
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
  - "NEVER suggest overly broad SSM document permissions"
  - "NEVER suggest disabling session logging"
  - "NEVER suggest sharing SSM documents publicly"
