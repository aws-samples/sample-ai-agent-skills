---
title: "A1 — Access Point Creation"
description: "Diagnose Object Lambda access point creation failures"
status: active
severity: HIGH
triggers: ["cannot create object lambda access point", "OLAP creation failed", "CreateAccessPointForObjectLambda error"]
owner: devops-agent
objective: "Resolve Object Lambda access point creation issues"
context: "OLAPs require a supporting standard S3 access point. The supporting AP must exist in the same region and account."
---

## Phase 1 — Triage

MUST:
- List existing OLAPs: `aws s3control list-access-points-for-object-lambda --account-id <account-id>`
- Verify supporting access point exists: `aws s3control get-access-point --account-id <account-id> --name <supporting-ap-name>`
- Check IAM permissions for s3:CreateAccessPointForObjectLambda

SHOULD:
- Verify the supporting access point is in the same region
- Check if the Lambda function ARN in the configuration is valid: `aws lambda get-function --function-name <function-arn>`

MAY:
- Review CloudTrail for CreateAccessPointForObjectLambda errors
- Check service quotas for Object Lambda access points per account

## Phase 2 — Remediate

MUST:
- Create supporting access point if missing: `aws s3control create-access-point --account-id <account-id> --name <ap-name> --bucket <bucket-name>`
- Fix Lambda function ARN in OLAP configuration
- Ensure IAM policy allows s3:CreateAccessPointForObjectLambda

SHOULD:
- Validate Lambda function exists and is in the same region
- Verify the transformation configuration specifies valid actions (GetObject, HeadObject, ListObjects)

MAY:
- Use CloudFormation or Terraform for repeatable OLAP creation
- Tag the OLAP for cost tracking and organization

## Common Issues

- symptoms: "CreateAccessPointForObjectLambda returns InvalidRequest — supporting access point not found"
  diagnosis: "The supporting S3 access point does not exist or is in a different region."
  resolution: "Create the supporting access point first: aws s3control create-access-point --account-id <id> --name <name> --bucket <bucket>."

- symptoms: "CreateAccessPointForObjectLambda returns InvalidArgument — invalid Lambda function ARN"
  diagnosis: "The Lambda function ARN is malformed or the function does not exist."
  resolution: "Verify the function exists: aws lambda get-function --function-name <arn>. Ensure the ARN includes the correct region and account."

## Safety Ratings

```
safety_ratings:
  - "list-access-points-for-object-lambda: GREEN — read-only enumeration"
  - "create-access-point-for-object-lambda: YELLOW — creates a new OLAP resource"
  - "create-access-point: YELLOW — creates a supporting access point"
  - "lambda get-function: GREEN — read-only function metadata"
```

## Escalation Conditions

- OLAP creation fails with internal service errors after fixing all configuration issues
- Service quota limit reached for Object Lambda access points in the account
- Supporting access point creation blocked by S3 Block Public Access settings at account level
- Lambda function is in a different account requiring cross-account configuration

## Data Sensitivity

- `aws s3control list-access-points-for-object-lambda`: LOW — returns OLAP metadata only
- `aws s3control get-access-point`: LOW — returns access point configuration
- `aws lambda get-function`: MEDIUM — may reveal function environment variables and VPC config
- `aws cloudtrail lookup-events`: MEDIUM — may reveal IAM principal identities

## Prohibited Actions

- NEVER delete an existing supporting access point that other OLAPs depend on
- NEVER create OLAPs with overly permissive policies (s3-object-lambda:* to *)
- NEVER hardcode AWS credentials in Lambda function configurations
- NEVER modify production OLAPs without verifying the change in a staging environment first

## Phase 3 — Rollback

1. If an OLAP was created with wrong configuration, delete it: `aws s3control delete-access-point-for-object-lambda --account-id <account-id> --name <olap-name>`
2. If a supporting access point was created unnecessarily, delete it: `aws s3control delete-access-point --account-id <account-id> --name <ap-name>` (only if no other OLAPs reference it)
3. If IAM policies were modified, revert to the previous policy version
4. Verify no active clients are using the deleted OLAP ARN
5. Document the rollback action and update the configuration management system

## Output Format

```yaml
root_cause: "olap_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation prerequisites"
  long_term: "Deploy OLAPs via IaC with validation"
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
  - command: "get-bucket-policy"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-bucket-acl"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"
  - command: "get-public-access-block"
    sensitivity: MEDIUM
    contains: "Service configuration and resource details"

## Prohibited Actions

prohibited_actions:
  - "NEVER suggest disabling S3 Block Public Access"
  - "NEVER suggest Principal: * without Condition keys"
  - "NEVER suggest removing bucket encryption"
