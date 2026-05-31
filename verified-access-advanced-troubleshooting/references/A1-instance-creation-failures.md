---
title: "A1 — Instance Creation Failures"
description: "Diagnose Verified Access instance creation failures"
status: active
severity: HIGH
triggers: ["cannot create verified access instance", "instance creation failed", "CreateVerifiedAccessInstance error"]
owner: devops-agent
objective: "Resolve Verified Access instance creation issues"
context: "A Verified Access instance is the top-level resource that groups trust providers, endpoint groups, and endpoints. Creation requires proper IAM permissions and trust provider configuration."
---

## Phase 1 — Triage

MUST:
- List existing instances: `aws ec2 describe-verified-access-instances --query 'VerifiedAccessInstances[*].{Id:VerifiedAccessInstanceId,Description:Description,TrustProviders:VerifiedAccessTrustProviders}'`
- Verify IAM permissions for ec2:CreateVerifiedAccessInstance
- Check CloudTrail for creation errors: `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateVerifiedAccessInstance --max-results 10`

SHOULD:
- Verify trust providers exist before attaching: `aws ec2 describe-verified-access-trust-providers`
- Check service quotas for Verified Access instances in the Region
- Review VPC and networking prerequisites

MAY:
- Check AWS Health Dashboard for service issues
- Review organization SCPs that may block Verified Access operations

## Phase 2 — Remediate

MUST:
- Ensure IAM principal has ec2:CreateVerifiedAccessInstance permission
- Create trust providers before creating the instance if attaching at creation time
- Verify the Region supports Verified Access

SHOULD:
- Use descriptive names and tags for the instance
- Enable logging configuration at creation time

MAY:
- Use CloudFormation or Terraform for repeatable deployments
- Configure WAF association during instance creation

## Common Issues

- symptoms: "CreateVerifiedAccessInstance returns UnauthorizedOperation"
  diagnosis: "IAM principal lacks ec2:CreateVerifiedAccessInstance permission."
  resolution: "Add ec2:CreateVerifiedAccessInstance to the IAM policy. Check for SCP restrictions."

## Safety Ratings

```
safety_ratings:
  - "describe-verified-access-instances: GREEN — read-only enumeration"
  - "CreateVerifiedAccessInstance: YELLOW — creates a new resource"
  - "AttachVerifiedAccessTrustProvider: YELLOW — modifies trust configuration"
  - "cloudtrail lookup-events: GREEN — read-only audit log review"
```

## Escalation Conditions

- Instance creation fails with internal service errors after IAM fixes
- Service quota limits reached and increase request is pending
- SCP restrictions require organizational policy changes
- Trust provider integration requires third-party vendor coordination

## Data Sensitivity

- `aws ec2 describe-verified-access-instances`: LOW — returns instance metadata
- `aws ec2 describe-verified-access-trust-providers`: MEDIUM — reveals trust provider configuration
- `aws cloudtrail lookup-events`: MEDIUM — reveals API call history and principal identities
- Trust provider credentials: HIGH — OIDC client secrets and provider configurations

## Prohibited Actions

- NEVER delete a Verified Access instance with active endpoints without migrating traffic first
- NEVER attach trust providers without verifying their configuration is correct
- NEVER create instances without planning the logging configuration
- NEVER expose trust provider credentials (OIDC client secrets) in logs or documentation

## Phase 3 — Rollback

1. If an instance was created in error, detach trust providers first: `aws ec2 detach-verified-access-trust-provider --verified-access-instance-id <id> --verified-access-trust-provider-id <tp-id>`
2. Delete the instance: `aws ec2 delete-verified-access-instance --verified-access-instance-id <id>`
3. If trust providers were created for the instance, delete them if no longer needed
4. Verify no DNS records point to the deleted instance's endpoints
5. Document the rollback and notify affected teams

## Output Format

```yaml
root_cause: "instance_creation — <specific_cause>"
evidence:
  - type: creation_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix creation issue"
  long_term: "Deploy via IaC with proper IAM and trust provider configuration"
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
  - "NEVER suggest overly broad IAM permissions as a fix"
  - "NEVER suggest disabling security controls for troubleshooting"
  - "NEVER suggest deleting resources without confirmation"
