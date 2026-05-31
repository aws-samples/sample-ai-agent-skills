---
title: "F1 — IAM Configuration"
description: "Diagnose IAM role, policy, and permission issues for FleetWise operations"
status: active
severity: HIGH
triggers: ["access denied", "IAM permission error", "role assumption failed", "FleetWise authorization error"]
owner: devops-agent
objective: "Resolve IAM configuration issues for FleetWise service and destination access"
context: "FleetWise requires IAM roles for service operations and data destinations. The service role allows FleetWise to write to S3/Timestream. Caller permissions control API access. Trust policies must allow iotfleetwise.amazonaws.com."
---

## Phase 1 — Triage

MUST:
- Check caller IAM permissions: `aws sts get-caller-identity`
- Verify FleetWise service role exists: `aws iam get-role --role-name <fleetwise-role>`
- Check service role trust policy allows FleetWise: `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'`

SHOULD:
- List role policies: `aws iam list-attached-role-policies --role-name <role>` and `aws iam list-role-policies --role-name <role>`
- Check for permission boundaries that may restrict access
- Verify no SCPs block FleetWise operations: check organization policies

MAY:
- Simulate permissions: `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names iotfleetwise:CreateCampaign s3:PutObject timestream:WriteRecords`
- Review CloudTrail for AccessDenied events
- Check for condition keys in policies that may restrict access

## Phase 2 — Remediate

MUST:
- Ensure caller has required iotfleetwise:* permissions for API operations
- Configure service role trust policy with iotfleetwise.amazonaws.com as trusted principal
- Add destination permissions to service role: s3:PutObject for S3, timestream:WriteRecords for Timestream

SHOULD:
- Use least-privilege policies — scope permissions to specific resources
- Add condition keys to trust policy for confused deputy protection (aws:SourceAccount, aws:SourceArn)
- Verify permission boundaries do not block required actions

MAY:
- Create separate roles for different FleetWise operations (management vs data delivery)
- Implement IAM policy versioning for change tracking
- Set up CloudTrail alerts for AccessDenied events on FleetWise resources

## Safety Ratings

```
safety_ratings:
  - "sts get-caller-identity: GREEN — read-only identity check"
  - "iam get-role: GREEN — read-only role inspection"
  - "iam list-attached-role-policies: GREEN — read-only policy listing"
  - "iam simulate-principal-policy: GREEN — read-only permission simulation"
  - "iam put-role-policy: YELLOW — modifies role permissions"
  - "iam update-assume-role-policy: RED — modifies trust relationships"
```

## Escalation Conditions

- SCP at organization level blocks FleetWise operations
- Permission boundary prevents adding required permissions
- Cross-account role assumption needed for multi-account FleetWise setup
- Service-linked role issues requiring AWS Support intervention

## Data Sensitivity

- `aws sts get-caller-identity`: LOW — returns caller account and principal
- `aws iam get-role`: MEDIUM — reveals trust policy and role configuration
- `aws iam list-attached-role-policies`: MEDIUM — reveals permission structure
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call patterns

## Prohibited Actions

- NEVER grant iotfleetwise:* with Resource: * in production — scope to specific resources
- NEVER modify trust policies without confused deputy protection conditions
- NEVER share IAM role credentials or access keys
- NEVER remove permissions from a role used by active campaigns without suspending campaigns first

## Phase 3 — Rollback

1. If role policy was modified incorrectly, restore previous version: `aws iam put-role-policy --role-name <role> --policy-name <policy> --policy-document file://previous-policy.json`
2. If trust policy was broken, restore: `aws iam update-assume-role-policy --role-name <role> --policy-document file://previous-trust.json`
3. If role was deleted, recreate with same name and policies
4. Verify FleetWise operations resume after IAM fixes
5. Test destination access by checking for new data delivery

## Common Issues

- symptoms: "FleetWise API calls return AccessDeniedException"
  diagnosis: "Caller IAM policy does not include required iotfleetwise:* actions."
  resolution: "Add iotfleetwise permissions to the caller's IAM policy. Use managed policy AmazonIoTFleetWiseFullAccess for testing, then scope down."

- symptoms: "Campaign created but data not delivered to destination"
  diagnosis: "Service role trust policy does not allow iotfleetwise.amazonaws.com, or role lacks destination write permissions."
  resolution: "Update trust policy to include iotfleetwise.amazonaws.com. Add s3:PutObject or timestream:WriteRecords to role policy."

## Output Format

```yaml
root_cause: "iam_configuration — <specific_cause>"
evidence:
  - type: iam_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix IAM permissions"
  long_term: "Implement least-privilege IAM policies with confused deputy protection"
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
