---
title: "C2 — Non-Compliant Resources"
description: "Diagnose and remediate resources flagged as non-compliant by detective controls"
status: active
severity: MEDIUM
triggers:
  - "Non-compliant"
  - "Config rule violation"
  - "Guardrail non-compliance"
owner: devops-agent
objective: "Identify non-compliant resources and plan remediation"
context: "Detective controls use AWS Config rules to evaluate resource compliance. Non-compliance means a resource violates the rule but the action was NOT blocked (detective controls don't prevent). Remediation is a separate action."
---

## Phase 1 — Triage

MUST:
- Identify the non-compliant control: `aws controltower list-enabled-controls --target-identifier <ou-arn>`
- Check Config compliance details: `aws configservice describe-compliance-by-config-rule --compliance-types NON_COMPLIANT`
- Get specific non-compliant resources: `aws configservice get-compliance-details-by-config-rule --config-rule-name <rule> --compliance-types NON_COMPLIANT`
- Identify the affected accounts and resources

SHOULD:
- Determine if the non-compliance is new or pre-existing
- Check if the resource was created before the control was enabled
- Assess the blast radius (how many accounts/resources affected)

MAY:
- Check CloudTrail for who created the non-compliant resource
- Review if auto-remediation is configured for the Config rule

## Phase 2 — Remediate

MUST:
- Fix the non-compliant resources (encrypt volumes, enable logging, etc.)
- Verify compliance after remediation: re-evaluate the Config rule

SHOULD:
- Enable a preventive control (SCP) to prevent future non-compliance
- Set up Config remediation actions for automatic fixes
- Notify account owners of non-compliant resources

MAY:
- Create a compliance dashboard for ongoing monitoring
- Implement proactive controls (CF hooks) to catch non-compliance at deployment time

## Common Issues

- symptoms: "Resources non-compliant for encryption control"
  diagnosis: "Unencrypted EBS volumes, S3 buckets, or RDS instances exist."
  resolution: "Encrypt the resources. Enable preventive SCP to block unencrypted resource creation."

- symptoms: "Non-compliance persists after fix"
  diagnosis: "Config rule hasn't re-evaluated yet."
  resolution: "Trigger re-evaluation: `aws configservice start-config-rules-evaluation --config-rule-names <rule>`"

- symptoms: "Many accounts non-compliant after enabling detective control"
  diagnosis: "Pre-existing resources created before the control was enabled."
  resolution: "Remediate existing resources. This is expected for detective controls on existing environments."

## Safety Ratings

```
safety_ratings:
  - "describe-compliance-by-config-rule: GREEN — read-only compliance status"
  - "get-compliance-details-by-config-rule: GREEN — read-only resource compliance"
  - "start-config-rules-evaluation: GREEN — triggers re-evaluation, no changes"
  - "remediate non-compliant resources (encrypt, enable logging): YELLOW — recoverable, resource-specific changes"
  - "enable preventive SCP to block future non-compliance: RED — can break existing workloads across all accounts in the OU"
```

## Escalation Conditions

- Fix requires modifying CT-managed SCPs
- Drift affects multiple OUs or accounts
- Non-compliance affects security-critical resources (encryption, logging)
- Remediation requires changes across many accounts simultaneously
- Fix requires re-registering OUs in production

## Data Sensitivity

- HIGH: SCP policy documents (expose security boundaries)
- HIGH: Organization account list (all account IDs and emails)
- MEDIUM: OU structure, control enablement status
- MEDIUM: Non-compliant resource details (resource IDs, configurations)

## Prohibited Actions

- "NEVER suggest directly modifying CT-managed CloudFormation stacks"
- "NEVER suggest directly editing CT-managed SCPs"
- "NEVER suggest deleting the Security OU or shared accounts"
- "NEVER suggest disabling CloudTrail organization trail"
- "NEVER suggest removing CT IAM roles"

## Phase 3 — Rollback

- If remediation of a resource causes service disruption, revert the specific resource change (e.g., remove encryption if it breaks an application)
- If a preventive SCP was enabled and blocks legitimate workloads, disable the control immediately
- "If OU re-registration fails, document the error and do NOT retry without investigating"
- "If landing zone update fails, do NOT attempt to roll back manually — contact AWS Support"
- Document the list of non-compliant resources and their current state before remediation

## Output Format

```yaml
root_cause: "non_compliance — <control_id> in <accounts>"
evidence:
  - type: config_compliance
    content: "<non-compliant resources and rules>"
severity: MEDIUM
mitigation:
  immediate: "Remediate non-compliant resources"
  long_term: "Enable preventive controls and auto-remediation"
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
