---
title: "C2 — Policy Configuration"
description: "Diagnose OLAP and supporting access point policy configuration issues"
status: active
severity: MEDIUM
triggers: ["policy misconfigured", "wrong policy", "policy not working", "cross-account OLAP access"]
owner: devops-agent
objective: "Resolve policy configuration issues for Object Lambda access points"
context: "Three policy layers control OLAP access: OLAP resource policy, supporting AP policy, and IAM identity policies. All must align."
---

## Phase 1 — Triage

MUST:
- Get OLAP resource policy: `aws s3control get-access-point-policy-for-object-lambda --account-id <account-id> --name <olap-name>`
- Get supporting access point policy: `aws s3control get-access-point-policy --account-id <account-id> --name <supporting-ap-name>`
- Get bucket policy: `aws s3api get-bucket-policy --bucket <bucket-name>`

SHOULD:
- Check S3 Block Public Access settings at account and bucket level
- Verify VPC endpoint policies if accessing from within a VPC
- Check for SCPs that may restrict s3-object-lambda actions

MAY:
- Use Access Analyzer to validate policies: `aws accessanalyzer validate-policy --policy-document file://policy.json --policy-type RESOURCE_POLICY`
- Review policy evaluation logic for explicit deny conflicts

## Phase 2 — Remediate

MUST:
- Align OLAP policy with intended access patterns
- Ensure supporting access point policy does not block OLAP access
- For cross-account: add principal ARN from the other account to the OLAP resource policy

SHOULD:
- Use condition keys to restrict access (aws:SourceVpc, aws:PrincipalOrgID)
- Validate policies with Access Analyzer before applying

MAY:
- Implement policy templates for consistent OLAP deployments
- Set up AWS Config rules for policy compliance monitoring

## Common Issues

- symptoms: "OLAP policy allows access but requests still fail"
  diagnosis: "Supporting access point policy is more restrictive than the OLAP policy."
  resolution: "Update supporting access point policy to allow the same principals as the OLAP policy."

- symptoms: "Cross-account access fails despite OLAP resource policy allowing the external account"
  diagnosis: "The external account's IAM policy does not grant s3-object-lambda:GetObject on the OLAP ARN."
  resolution: "Both the OLAP resource policy AND the caller's IAM policy must allow access."

## Safety Ratings

```
safety_ratings:
  - "get-access-point-policy-for-object-lambda: GREEN — read-only"
  - "get-access-point-policy: GREEN — read-only"
  - "get-bucket-policy: GREEN — read-only"
  - "put-access-point-policy-for-object-lambda: RED — modifies access control"
  - "accessanalyzer validate-policy: GREEN — read-only validation"
```

## Escalation Conditions

- Policy evaluation results in unexpected denials despite correct configuration
- SCP at organization level blocks s3-object-lambda actions
- VPC endpoint policy requires changes managed by a different team
- Cross-account access requires trust relationship changes in a regulated environment

## Data Sensitivity

- `aws s3control get-access-point-policy-for-object-lambda`: MEDIUM — reveals access control
- `aws s3control get-access-point-policy`: MEDIUM — reveals access control
- `aws s3api get-bucket-policy`: MEDIUM — reveals bucket access control
- `aws accessanalyzer validate-policy`: LOW — validates policy syntax only

## Prohibited Actions

- NEVER apply a policy with Principal: "*" without a restrictive Condition block
- NEVER remove Deny statements from policies without understanding their purpose
- NEVER modify bucket policies that affect other access points or direct bucket access
- NEVER grant cross-account access without explicit approval from the security team

## Phase 3 — Rollback

1. Restore previous OLAP policy: `aws s3control put-access-point-policy-for-object-lambda --account-id <account-id> --name <olap-name> --policy file://previous-policy.json`
2. Restore previous supporting AP policy: `aws s3control put-access-point-policy --account-id <account-id> --name <ap-name> --policy file://previous-policy.json`
3. Restore previous bucket policy if modified: `aws s3api put-bucket-policy --bucket <bucket-name> --policy file://previous-policy.json`
4. Verify all access patterns work correctly after rollback
5. Document the policy change attempt and lessons learned

## Output Format

```yaml
root_cause: "policy_configuration — <specific_cause>"
evidence:
  - type: policy_analysis
    content: "<policy evaluation details>"
severity: MEDIUM
mitigation:
  immediate: "Fix policy alignment"
  long_term: "Implement policy-as-code with cross-layer validation"
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
