---
title: "A1 — Fleet Provisioning Failures"
description: "Diagnose fleet provisioning template and claim-based provisioning failures"
status: active
severity: HIGH
triggers: ["fleet provisioning failed", "claim certificate rejected", "provisioning template error", "RegisterThing failed"]
owner: devops-agent
objective: "Resolve fleet provisioning failures and restore device onboarding"
context: "Fleet provisioning supports claim-based and trusted-user flows. Claim-based uses a shared claim certificate for bootstrap. Trusted-user uses an authorized admin."
---

## Phase 1 — Triage

MUST:
- List provisioning templates: `aws iot list-provisioning-templates --query 'templates[*].{Name:templateName,Enabled:enabled,Type:type}'`
- Describe the failing template: `aws iot describe-provisioning-template --template-name <name>`
- Check claim certificate status: `aws iot describe-certificate --certificate-id <cert-id> --query '{Status:certificateDescription.status,Arn:certificateDescription.certificateArn}'`

SHOULD:
- Verify the provisioning role permissions: `aws iam get-role --role-name <provisioning-role> --query 'Role.{Arn:Arn,AssumeRolePolicyDocument:AssumeRolePolicyDocument}'`
- Check IoT policy attached to claim certificate: `aws iot list-attached-policies --target <cert-arn>`
- Review CloudWatch Logs for provisioning errors: `aws logs filter-log-events --log-group-name AWSIotLogsV2 --filter-pattern "fleet provisioning"`

MAY:
- Check CloudTrail for RegisterThing API calls: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RegisterThing --max-results 10`
- Verify pre-provisioning hook Lambda function: `aws lambda get-function --function-name <hook-function>`

## Phase 2 — Remediate

MUST:
- Ensure provisioning template is enabled: `aws iot update-provisioning-template --template-name <name> --enabled`
- Verify claim certificate is ACTIVE: `aws iot update-certificate --certificate-id <id> --new-status ACTIVE`
- Ensure provisioning role trust policy allows iot.amazonaws.com to assume it

SHOULD:
- Validate template body JSON syntax and parameter references
- Ensure IoT policy on claim certificate allows iot:Connect, iot:Publish on $aws/certificates/create/* and $aws/provisioning-templates/<name>/provision/*
- Test pre-provisioning hook Lambda independently

MAY:
- Enable IoT Core logging for detailed diagnostics: `aws iot set-v2-logging-options --role-arn <log-role-arn> --default-log-level DEBUG`
- Create a test device to validate the provisioning flow end-to-end

## Safety Ratings

```
safety_ratings:
  - "list-provisioning-templates: GREEN — read-only enumeration"
  - "describe-provisioning-template: GREEN — read-only inspection"
  - "update-provisioning-template --enabled: YELLOW — enables provisioning for all devices with claim cert"
  - "update-certificate --new-status ACTIVE: YELLOW — activates certificate for authentication"
  - "set-v2-logging-options --default-log-level DEBUG: YELLOW — increases logging volume and cost"
```

## Escalation Conditions

- Provisioning hook Lambda consistently rejects valid devices
- Claim certificate compromised or needs rotation across fleet
- Provisioning template changes required in production affecting active device onboarding
- IAM role trust policy changes blocked by SCP or permissions boundary

## Data Sensitivity

- `aws iot describe-provisioning-template`: MEDIUM — contains template body with resource patterns
- `aws iot describe-certificate`: LOW — returns certificate metadata, not private key
- `aws iam get-role`: LOW — returns role metadata and trust policy
- `aws logs filter-log-events`: MEDIUM — may contain device identifiers and error details

## Prohibited Actions

- NEVER share or log claim certificate private keys
- NEVER disable a provisioning template in production without confirming no active onboarding
- NEVER delete a claim certificate that is actively used by devices in the field
- NEVER modify the provisioning role permissions without understanding the blast radius across all templates using it

## Phase 3 — Rollback

1. If template was incorrectly modified, restore from version history or IaC source
2. If claim certificate was deactivated, reactivate: `aws iot update-certificate --certificate-id <id> --new-status ACTIVE`
3. If provisioning role was modified, restore previous policy version: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://previous-policy.json`
4. If logging level was changed, restore: `aws iot set-v2-logging-options --role-arn <arn> --default-log-level INFO`
5. Verify device onboarding works by testing with a single device

## Common Issues

- symptoms: "MQTT connection rejected during provisioning with claim certificate"
  diagnosis: "Claim certificate is INACTIVE or revoked, or IoT policy does not allow required MQTT topics."
  resolution: "Activate certificate and attach policy allowing iot:Connect and iot:Publish on provisioning topics."

- symptoms: "Pre-provisioning hook returns DENY for all devices"
  diagnosis: "Lambda function logic error or missing device attributes in the provisioning parameters."
  resolution: "Test Lambda with sample event payload. Verify device sends required parameters in provisioning request."

## Output Format

```yaml
root_cause: "fleet_provisioning — <specific_cause>"
evidence:
  - type: provisioning_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix provisioning configuration"
  long_term: "Implement provisioning monitoring and alerting"
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
  - command: "describe-instances"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-console-output"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "ssm send-command"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest 0.0.0.0/0 inbound security group rules as a fix"
  - "NEVER suggest disabling instance metadata service"
  - "NEVER terminate instances without confirmation"
