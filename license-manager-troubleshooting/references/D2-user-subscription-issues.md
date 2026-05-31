---
title: "D2 — User Subscription Issues"
description: "Diagnose user-based subscription issues (Visual Studio, Office)"
status: active
severity: HIGH
triggers: ["user subscription error", "visual studio license", "user-based license", "AD integration"]
owner: devops-agent
objective: "Resolve user-based subscription configuration and access issues"
context: "User-based subscriptions require Active Directory integration, specific VPC configuration, and proper IAM roles. Used for per-user software like Microsoft Visual Studio."
---

## Phase 1 — Triage

MUST:
- Check user subscription settings: `aws license-manager-user-subscriptions list-user-associations --instance-id <id>`
- Verify Active Directory connectivity: `aws ds describe-directories --query 'DirectoryDescriptions[*].{Id:DirectoryId,Name:Name,Status:Status}'`
- Check VPC DNS configuration: `aws ec2 describe-vpc-attribute --vpc-id <vpc-id> --attribute enableDnsSupport`

SHOULD:
- Verify IAM roles for License Manager user subscriptions
- Check network connectivity between VPC and directory service
- List registered instances: `aws license-manager-user-subscriptions list-instances`

MAY:
- Check AD user accounts and group memberships
- Review security group rules for AD connectivity

## Phase 2 — Remediate

MUST:
- Set up Active Directory (AWS Managed AD or AD Connector): `aws ds describe-directories`
- Configure VPC with DNS resolution pointing to AD
- Create required IAM roles for user subscription management

SHOULD:
- Register instances for user subscriptions: `aws license-manager-user-subscriptions register-identity-provider --identity-provider '{"ActiveDirectoryIdentityProvider":{"DirectoryId":"<dir-id>"}}'`
- Associate users with instances

MAY:
- Set up automated user provisioning
- Configure SSO for seamless user access

## Safety Ratings

```
safety_ratings:
  - "license-manager-user-subscriptions list-user-associations: GREEN — read-only listing"
  - "ds describe-directories: GREEN — read-only directory check"
  - "license-manager-user-subscriptions register-identity-provider: YELLOW — configures AD integration"
  - "ec2 describe-vpc-attribute: GREEN — read-only VPC check"
```

## Escalation Conditions

- Active Directory connectivity fails persistently
- User subscription licensing does not match AD user count
- VPC configuration changes affect other workloads
- Identity provider registration fails with internal errors

## Data Sensitivity

- Active Directory details: HIGH — contains identity infrastructure information
- User associations: HIGH — reveals user-to-resource mappings
- VPC configuration: MEDIUM — reveals network architecture
- IAM roles: MEDIUM — reveals permission structure

## Prohibited Actions

- NEVER expose Active Directory credentials in logs or outputs
- NEVER modify AD configuration without directory admin approval
- NEVER change VPC DNS settings without understanding impact on other services
- NEVER assign user subscriptions without verifying license availability

## Phase 3 — Rollback

1. Disassociate users from instances if incorrectly assigned
2. Deregister identity provider if configured incorrectly
3. Revert VPC DNS changes if they affected other services
4. Remove IAM roles created for user subscriptions if not needed
5. Document user subscription changes for compliance

## Common Issues

- symptoms: "User subscription association fails with DirectoryNotReachable"
  diagnosis: "VPC DNS not configured to resolve AD domain or security groups blocking AD ports."
  resolution: "Configure VPC DNS to point to AD. Open ports 389 (LDAP), 636 (LDAPS), 88 (Kerberos) in security groups."

## Output Format

```yaml
root_cause: "user_subscription — <specific_cause>"
evidence:
  - type: subscription_error
    content: "<error details>"
severity: HIGH
mitigation:
  immediate: "Fix user subscription configuration"
  long_term: "Automate user provisioning and monitoring"
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
